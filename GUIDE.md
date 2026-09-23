# Enterprise Knowledge Assistant — Complete Guide

> A multi-agent RAG system that answers employee queries using internal documents and provides answers with citations.

---

## Part 1: Project Task Breakdown (Industry Style)

| Task | Name | Goal |
|------|------|------|
| 1 | Project Setup | Create folder structure, requirements, .env |
| 2 | Sample Documents | Add company documents to search over |
| 3 | Document Loader | Read & split files into chunks |
| 4 | Vector Store (FAISS) | Convert chunks to vectors, save to disk |
| 5 | Retriever | Search vectors by similarity |
| 6 | Router Agent | Classify query: needs docs or general? |
| 7 | RAG Agent | Retrieve docs + generate grounded answer |
| 8 | Citation Agent | Format sources for the answer |
| 9 | LangGraph Workflow | Wire all agents together |
| 10 | FastAPI Backend | Expose everything as REST API |
| 11 | Streamlit Frontend | Build the chat UI |

---

## Part 2: Step-by-Step Build

---

### Step 1: Project Setup

**What:** Create the folder structure and install dependencies.

**Why:** Real projects always start with structure. Just like you build a house's frame before the walls, you set up the project skeleton first.

**Files created:**
- `requirements.txt` — lists every Python library we need
- `.env` — holds secret API keys (never commit this to git!)

**How to run this step:**
```bash
cd enterprise_knowledge_assistant
pip install -r requirements.txt
```

**Expected output:** All packages install without errors.

---

### Step 2: Sample Documents (Knowledge Base)

**What:** We created 3 realistic company documents in `backend/data/sample_docs/`:
- `company_policy.txt` — WFH, leave, expenses, code of conduct
- `employee_handbook.txt` — Onboarding, IT systems, communication
- `benefits_guide.txt` — Health insurance, 401k, equity, perks

**Why:** The AI needs something to READ. Without documents, RAG doesn't work. These documents are the "knowledge" in our Knowledge Assistant.

**Real world:** Companies would upload their actual HR documents, policy PDFs, etc.

---

### Step 3: Document Loader (`backend/rag/document_loader.py`)

**What:** Reads documents and splits them into chunks.

**Why does splitting matter?** Imagine searching a 100-page document. Instead of reading all 100 pages, you cut it into 100 separate 1-page sections. Now you can find the EXACT page that answers a question.

```
Original document (5000 words)
         ↓ split
Chunk 1 (1000 chars)  ← "Annual Leave: 15 days for..."
Chunk 2 (1000 chars)  ← "Sick Leave: 10 days per year..."
Chunk 3 (1000 chars)  ← "Emergency Leave: 5 days..."
         ↓
Each chunk can be independently searched!
```

**Key function:**
```python
# Splits text into overlapping chunks
RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
```

The `chunk_overlap=200` means neighboring chunks share 200 characters. This prevents losing context at boundaries.

---

### Step 4: Vector Store (`backend/rag/embeddings.py`)

**What:** Converts text chunks into vectors (lists of numbers) and stores them in FAISS.

**Why use vectors?** Computers can't "understand" text directly. But they can compare numbers. Embeddings convert text into numbers in a way that **similar meanings → similar numbers**.

```
"How many leave days?" → [0.23, -0.45, 0.89, ...]   ← query vector
"Annual Leave: 15 days" → [0.24, -0.44, 0.91, ...]  ← very close!
"Office parking policy" → [0.91, 0.32, -0.12, ...]  ← very different
```

FAISS finds the closest vectors in milliseconds, even with millions of vectors.

**The save/load trick:** We save the vector store to disk (`data/vectorstore/`) so we don't pay for OpenAI API calls every time the server restarts.

---

### Step 5: Retriever (`backend/rag/retriever.py`)

**What:** A wrapper around FAISS that makes retrieval easy to use.

**How retrieval works:**
```
User Query → OpenAI Embedding API → Query Vector
                                        ↓
                                   FAISS Search
                                        ↓
                               Top 4 matching chunks
                                        ↓
                               Return as Documents
```

**Key method:**
```python
def retrieve(self, query: str) -> List[Document]:
    return self.vectorstore.similarity_search(query, k=self.top_k)
```

One API call converts the query to a vector, FAISS does the rest locally.

---

### Step 6: Router Agent (`backend/agents/router_agent.py`)

**What:** The first agent. Reads the user's question and decides which path to take.

**Why a router?** Not all questions need document search. This saves cost and improves accuracy:
- "How many leave days?" → needs company docs → send to RAG Agent
- "What is the capital of France?" → general knowledge → send to General Agent

**How it works (simple):**
```
User Query → LLM with routing prompt → "rag" or "general"
```

The prompt is carefully written to give the LLM clear rules:
```
If question is about company policies/benefits/HR → return "rag"
If question is general knowledge → return "general"
```

---

### Step 7: RAG Agent (`backend/agents/rag_agent.py`)

**What:** The most important agent. It retrieves documents AND generates the answer.

**The RAG loop (this is the heart of the system):**
```
1. Get user query
2. Retrieve top 4 relevant chunks from FAISS
3. Build a prompt:
   "Here are the relevant documents: [chunks...]
    Answer this question: [query]"
4. Send to LLM → get grounded answer
5. Return answer + the retrieved documents
```

**Why is this better than just asking the LLM?**
- LLM alone might make up (hallucinate) leave policies
- RAG grounds the answer in YOUR actual documents
- Answer is always traceable to a source

There's also a **General Agent** (in the same file) for non-document queries. It uses the LLM's built-in knowledge.

---

### Step 8: Citation Agent (`backend/agents/citation_agent.py`)

**What:** Takes the retrieved documents and formats them into clean citations.

**Why citations matter in enterprise:**
- Employees can verify the information
- Compliance and audit trail
- Builds trust ("I can see where this came from")

**Output format:**
```json
{
  "id": 1,
  "source": "company_policy.txt",
  "preview": "Annual Leave: 0-2 years of service: 15 days paid leave...",
  "full_text": "..."
}
```

---

### Step 9: LangGraph Workflow (`backend/agents/graph.py`)

**What:** Wires all 4 agents into a single automated workflow.

**Architecture diagram:**
```
[User Query]
     ↓
[Router Agent] ─── asks LLM: "rag or general?"
     ↓         ↓
[RAG Agent]  [General Agent]
     ↓
[Citation Agent]
     ↓
[Final Answer + Citations]
```

**LangGraph key concepts:**
- **State**: A dictionary that ALL agents share and update
- **Nodes**: Your agent functions
- **Edges**: Connections between agents
- **Conditional Edges**: "Go to RAG if route=='rag', else go to General"

**The state flows like a baton in a relay race:**
```python
state = {"query": "How many leave days?", "route": None, "answer": None, "citations": []}
# Router fills:   state["route"] = "rag"
# RAG fills:      state["answer"] = "15 days for..."
# Citation fills: state["citations"] = [{"source": "company_policy.txt"...}]
```

---

### Step 10: FastAPI Backend (`backend/main.py` + `backend/api/routes.py`)

**What:** Exposes everything as a REST API that the frontend can call.

**4 endpoints:**

| Endpoint | Method | What it does |
|----------|--------|-------------|
| `/api/health` | GET | Is the server alive? |
| `/api/query` | POST | Send question, get answer |
| `/api/ingest` | POST | Upload a new document |
| `/api/documents` | GET | List all documents |

**Startup flow (automatic when server starts):**
```
Load documents → Create/load FAISS → Initialize LLM → Build agent graph → Ready!
```

CORS middleware is added so Streamlit (port 8501) can talk to FastAPI (port 8000).

---

### Step 11: Streamlit Frontend (`frontend/app.py`)

**What:** A chat interface that talks to the FastAPI backend.

**Features:**
- Chat messages (like ChatGPT)
- Collapsible citation cards under each answer
- Color-coded badges showing which agent handled the query (RAG or General)
- Sidebar with document list, upload button, sample questions
- Live health status indicator

**How Streamlit works:**
```
User types → st.chat_input captures it
         → requests.post() sends to FastAPI
         → gets JSON response back
         → st.markdown() displays the answer
         → st.expander() shows citations
```

---

## Part 3: Final Project Structure

```
enterprise_knowledge_assistant/
│
├── backend/
│   ├── __init__.py                 # Makes backend a Python package
│   ├── main.py                     # FastAPI app + startup logic
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes.py               # /health, /query, /ingest, /documents
│   │
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── document_loader.py      # Read files, split into chunks
│   │   ├── embeddings.py           # Convert chunks to vectors (FAISS)
│   │   └── retriever.py            # Search for relevant chunks
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── graph.py                # LangGraph workflow definition
│   │   ├── router_agent.py         # Classifies: rag or general?
│   │   ├── rag_agent.py            # Retrieves + generates answer
│   │   └── citation_agent.py       # Formats source citations
│   │
│   ├── utils/
│   │   ├── __init__.py
│   │   └── helpers.py              # Shared utility functions
│   │
│   └── data/
│       ├── sample_docs/            # Company documents (.txt, .pdf)
│       │   ├── company_policy.txt
│       │   ├── employee_handbook.txt
│       │   └── benefits_guide.txt
│       └── vectorstore/            # FAISS index (auto-created)
│           ├── index.faiss
│           └── index.pkl
│
├── frontend/
│   └── app.py                      # Streamlit chat UI
│
├── .env                            # Your API keys (DO NOT commit to git)
├── .env.example                    # Template for .env
├── requirements.txt                # pip install -r requirements.txt
├── README.md                       # Quick start
└── GUIDE.md                        # This file
```

---

## Part 4: Architecture Explanation

### How RAG Works in This Project

```
                        ┌─────────────────────────┐
                        │   Company Documents      │
                        │  (company_policy.txt,    │
                        │   employee_handbook.txt) │
                        └───────────┬─────────────┘
                                    │ Step 1: Load & Split
                                    ▼
                        ┌─────────────────────────┐
                        │   Document Chunks        │
                        │  (1000-char pieces)      │
                        └───────────┬─────────────┘
                                    │ Step 2: Embed (OpenAI API)
                                    ▼
                        ┌─────────────────────────┐
                        │   FAISS Vector Store     │
                        │  (numbers saved on disk) │
                        └─────────────────────────┘
                                    ↑↓ Search
User Question ──embed──→ [query vector] ──→ top 4 matches
                                               │
                                               ▼
                                    ┌─────────────────────┐
                                    │  LLM with Context   │
                                    │ "Based on docs...   │
                                    │  The answer is..."  │
                                    └─────────────────────┘
```

### How Agents Interact

Each agent is like a specialist worker:

1. **Router Agent** — The receptionist. Decides who handles the request.
2. **RAG Agent** — The researcher. Digs through documents and generates answer.
3. **General Agent** — The generalist. Answers from built-in LLM knowledge.
4. **Citation Agent** — The editor. Adds proper source references.

They communicate through **shared state** (a Python dictionary that all agents read and write).

### Data Flow: UI → API → RAG → Response

```
Streamlit UI (port 8501)
  └─ User types: "How many leave days?"
  └─ requests.post("http://localhost:8000/api/query", json={"query": "..."})
        ↓
FastAPI (port 8000)
  └─ routes.py receives POST /api/query
  └─ calls run_agent(graph, query)
        ↓
LangGraph Graph
  └─ Router Agent  → route = "rag"
  └─ RAG Agent     → retrieves 4 chunks, generates answer
  └─ Citation Agent → formats 4 citations
        ↓
FastAPI Response (JSON)
  └─ {"answer": "You get 15 days...", "citations": [...], "route": "rag"}
        ↓
Streamlit UI
  └─ Displays answer in chat bubble
  └─ Shows collapsible citation cards
```

---

## Part 5: How to Run

### Step 1: Setup Environment

```bash
# Navigate into the project folder
cd enterprise_knowledge_assistant

# Install all dependencies (takes 2-5 minutes on first run)
pip install -r requirements.txt
```

### Step 2: Configure API Key

Open the `.env` file and replace the placeholder with your real OpenAI key:

```
OPENAI_API_KEY=sk-...your-real-key-here...
```

Get your API key from: https://platform.openai.com/api-keys

### Step 3: Run the Backend (Terminal 1)

```bash
# From inside the enterprise_knowledge_assistant/ folder
uvicorn backend.main:app --reload --port 8000
```

You will see startup logs like:
```
[Startup] Step 1: Loading documents from 'backend/data/sample_docs'...
[Startup] Step 2: Initializing vector store...
[Startup] Step 3: Initializing Language Model...
[Startup] Step 5: Building multi-agent workflow...
SERVER READY! All components initialized.
API docs: http://localhost:8000/docs
```

### Step 4: Run the Frontend (Terminal 2)

```bash
# Open a NEW terminal window
streamlit run frontend/app.py
```

Browser opens automatically at `http://localhost:8501`

### Step 5: Test the API (Optional)

Open `http://localhost:8000/docs` for interactive Swagger UI — test all endpoints in the browser.

Or use curl:
```bash
# Ask a question
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"query": "How many leave days do I get?"}'

# Check server health
curl http://localhost:8000/api/health

# List documents
curl http://localhost:8000/api/documents
```

---

## Part 6: Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `OPENAI_API_KEY not set` | Missing .env | Edit `.env`, add your real key |
| `Connection refused` | Backend not running | Run `uvicorn backend.main:app --reload` |
| `No module named X` | Missing package | Run `pip install -r requirements.txt` |
| `No documents found` | Empty sample_docs | Check `backend/data/sample_docs/` has .txt files |
| `Rate limit exceeded` | Too many OpenAI calls | Wait 1 minute, try again |
| `allow_dangerous_deserialization` error | Old LangChain version | Update: `pip install -U langchain-community` |

---

## Part 7: Key Concepts You Learned

| Concept | File Where You Built It | Why It Matters |
|---------|------------------------|----------------|
| **RAG** | `agents/rag_agent.py` | Grounds LLM in real documents, stops hallucination |
| **Embeddings** | `rag/embeddings.py` | Converts text to searchable numbers |
| **Vector Search** | `rag/retriever.py` | Finds similar documents instantly |
| **Multi-Agent System** | `agents/graph.py` | Specialization gives better results |
| **LangGraph** | `agents/graph.py` | Manages agent coordination and state |
| **FastAPI** | `main.py`, `api/routes.py` | Production-grade REST API |
| **Streamlit** | `frontend/app.py` | Python-only web UI, no HTML/CSS needed |
| **Citations** | `agents/citation_agent.py` | Trust, verifiability, compliance |
| **FAISS** | `rag/embeddings.py` | Fast vector search, runs locally |
| **Pydantic Models** | `api/routes.py` | Automatic request/response validation |

---

## Part 8: How to Add Your Own Documents

1. Drop any `.txt` or `.pdf` file into `backend/data/sample_docs/`
2. **Delete the old vector store** (so it rebuilds with new docs):
   ```bash
   rm -rf backend/data/vectorstore/
   ```
3. Restart the backend — it will automatically re-index everything

OR use the UI: click **"Upload New Document"** in the Streamlit sidebar.

---

## Part 9: Project Tech Stack Summary

```
┌─────────────────────────────────────────────────────┐
│                  FRONTEND LAYER                      │
│              Streamlit (Python UI)                   │
│         Chat interface + File upload                 │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP (requests library)
                       ▼
┌─────────────────────────────────────────────────────┐
│                   API LAYER                          │
│            FastAPI (REST endpoints)                  │
│      /query  /ingest  /documents  /health            │
└──────────────────────┬──────────────────────────────┘
                       │ Python function calls
                       ▼
┌─────────────────────────────────────────────────────┐
│               AGENT ORCHESTRATION LAYER              │
│          LangGraph (multi-agent workflow)            │
│   Router Agent → RAG Agent → Citation Agent          │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
           ▼                          ▼
┌──────────────────┐      ┌───────────────────────────┐
│  KNOWLEDGE LAYER │      │        LLM LAYER           │
│  FAISS + OpenAI  │      │   OpenAI GPT-3.5/GPT-4    │
│  Embeddings      │      │  (answers + routing)       │
│  Vector Search   │      └───────────────────────────┘
└──────────────────┘
```
