I checked your project. The pipeline is already written and your repo is in sync with GitHub, so what's left is setup in AWS, on the EC2 instance and in GitHub.

Two things to know first:
- You don't need a webhook. GitHub has a "self-hosted runner", a small agent you install on EC2. The runner opens an outgoing connection to GitHub and waits for jobs, so you don't open any extra port and there's no webhook to set up. Your aws.yaml already uses one (runs-on: self-hosted).
- Never paste AWS keys into the .yml file. It already reads them from GitHub's encrypted store using ${{ secrets.AWS_ACCESS_KEY_ID }}. You only add them in the GitHub UI (Step 4).

How it works: you push to main → GitHub's server builds the two Docker images → they're pushed to Amazon ECR (AWS's private image store) → the runner on EC2 pulls them and restarts the containers.

I made one small change: I added workflow_dispatch: to .github/workflows/aws.yaml. It gives you a Run workflow button so you can start the pipeline by hand. I haven't committed it yet.

---

Step 1 — AWS: create ECR repositories

AWS Console → ECR → Create repository (Private). Create two:
- eka-backend
- eka-frontend

Note your region (for example ap-south-1).

Step 2 — AWS: create an IAM user for GitHub

1. IAM → Users → Create user, name it github-actions, and don't give it console access.
2. Permissions → Attach policies directly → choose AmazonEC2ContainerRegistryFullAccess.
3. Open the user → Security credentials → Create access key → choose "Application running outside AWS".
4. Copy the Access key ID and Secret access key. The secret is shown only once.

Step 3 — Prepare the EC2 instance

Security group (inbound rules):

┌──────┬──────────────────┬──────────────┐
│ Port │      Source      │     Why      │
├──────┼──────────────────┼──────────────┤
│ 22   │ My IP            │ SSH          │
├──────┼──────────────────┼──────────────┤
│ 8501 │ 0.0.0.0/0        │ Streamlit UI │
├──────┼──────────────────┼──────────────┤
│ 8000 │ My IP (optional) │ API docs     │
└──────┴──────────────────┴──────────────┘

Use at least t3.small with a 20 GB disk. The images are big (LangChain, FAISS).

SSH in:
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>      # use ec2-user@ on Amazon Linux

Install Docker (Ubuntu):
sudo apt-get update
curl -fsSL https://get.docker.com | sudo sh     # installs Docker + the compose plugin
sudo usermod -aG docker $USER                   # lets you run docker without sudo
exit                                            # log out and SSH back in so the group applies
docker --version && docker compose version      # both should print a version
<details><summary>Using Amazon Linux 2023 instead?</summary>

sudo dnf install -y docker && sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
</details>

Step 4 — GitHub: add the secrets

Repo AI-Engineer-Course/Enterprise_Knowledg_Assistant → Settings → Secrets and variables → Actions → New repository secret. Add these six. The names must match exactly:

┌───────────────────────┬─────────────────┐
│         Name          │      Value      │
├───────────────────────┼─────────────────┤
│ AWS_ACCESS_KEY_ID     │ from Step 2     │
├───────────────────────┼─────────────────┤
│ AWS_SECRET_ACCESS_KEY │ from Step 2     │
├───────────────────────┼─────────────────┤
│ AWS_DEFAULT_REGION    │ e.g. ap-south-1 │
├───────────────────────┼─────────────────┤
│ ECR_REPO_BACKEND      │ eka-backend     │
├───────────────────────┼─────────────────┤
│ ECR_REPO_FRONTEND     │ eka-frontend    │
├───────────────────────┼─────────────────┤
│ OPENAI_API_KEY        │ your OpenAI key │
└───────────────────────┴─────────────────┘

You need admin rights on the repo to see Settings. It's in the AI-Engineer-Course org, so ask the owner if the tab is missing.

Step 5 — GitHub: connect EC2 as a self-hosted runner (the "auth" step)

1. Repo → Settings → Actions → Runners → New self-hosted runner → pick Linux, x64.
2. GitHub shows commands with a one-time token. That token is the authentication between EC2 and GitHub. Run them on EC2. They look like this, but copy the real ones from the page:
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-X.Y.Z.tar.gz -L https://github.com/actions/runner/releases/download/...
tar xzf ./actions-runner-linux-x64-*.tar.gz
./config.sh --url https://github.com/AI-Engineer-Course/Enterprise_Knowledg_Assistant --token <TOKEN>
   Press Enter to accept the defaults for the questions.
3. Install it as a service so it keeps running after you log out or reboot:
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status        # should say "active (running)"
4. Back in GitHub → Settings → Runners, you should see it with a green Idle status.

Step 6 — Deploy

From your Mac:
cd ~/Desktop/Enterprise_Knowledg_Assistant_main_2
git add .github/workflows/aws.yaml
git commit -m "Add manual trigger to deploy workflow"
git push origin main
The push starts the pipeline. Watch it in the repo's Actions tab. Continuous-Integration runs first (build and push), then Continuous-Deployment (on EC2). Later you can also re-run it with Actions → Run workflow.

Step 7 — Check it worked

- Open http://<EC2-PUBLIC-IP>:8501 in your browser to see the chat UI.
- On EC2, docker ps should show 2 containers, with the backend marked healthy.
- API docs (if you opened port 8000): http://<EC2-PUBLIC-IP>:8000/docs

Common failures:
- permission denied ... docker.sock: the runner started before you were added to the docker group. Fix it with sudo ./svc.sh stop && sudo ./svc.sh start.
- repository does not exist: the ECR repo names don't match the secrets.
- Deploy job stuck on "Waiting for a runner": the runner is offline. Check sudo ./svc.sh status on EC2.

I can create the two ECR repositories from your Mac with the AWS CLI if you want, but I'll show you the command before running it.
