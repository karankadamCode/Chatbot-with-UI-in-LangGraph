# LangGraph Multi-Utility Chatbot (PDF RAG + Tools + SQLite Memory + LangSmith)

A Streamlit-based multi-utility chatbot built using **LangGraph** + **LangChain**, with:
- **Tool calling** (Web Search, Calculator, Stock API, PDF RAG)
- **Thread-scoped PDF RAG** (each chat thread can index its own PDF using FAISS)
- **SQLite checkpointing** (persistent conversation memory per thread)
- **LangSmith observability** (tracing runs, tool calls, node transitions, errors)

---

## Architecture (High Level)

### What happens in a chat turn?
1. User sends a message from the UI with `thread_id`
2. `chat_node` calls the LLM (tools enabled)
3. If the LLM requests a tool call → LangGraph routes to `ToolNode`
4. Tool runs (search / calculator / stock / rag)
5. Tool output returns to `chat_node`
6. LLM final response is generated
7. Conversation state is checkpointed to SQLite
8. Entire run is traced to LangSmith (when enabled)

### Thread-scoped PDF RAG
- Upload a PDF in the sidebar
- The backend indexes the PDF into a FAISS vector store
- Retriever is stored per `thread_id` so PDFs do not mix across threads

---

## Folder Structure (Recommended)

```text
.
├── langraph_rag_backend.py        # Backend: LangGraph graph + tools + ingestion + SQLite checkpointer
├── streamlit_app.py               # Streamlit UI
├── requirements.txt               # Python dependencies
├── .env                           # Environment variables (NOT committed)
└── chatbot.db                     # SQLite DB created at runtime

---
### LangGraph PDF Chatbot — Architecture Flowchart (with LangSmith Observability)

┌──────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENT / UI                                 │
│                           (Streamlit / API Caller)                            │
└──────────────────────────────────────────────────────────────────────────────┘
                 │
                 │ user message + config { thread_id } + optional PDF upload
                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           SESSION / THREAD MANAGER                            │
│   - thread_id generated / selected                                             │
│   - message_history maintained                                                 │
│   - ingested_docs tracked per thread                                           │
└──────────────────────────────────────────────────────────────────────────────┘
         │                                      │
         │ PDF Upload                            │ Chat Turn
         ▼                                      ▼
┌──────────────────────────────┐        ┌──────────────────────────────────────┐
│        PDF INGESTION          │        │           LANGGRAPH GRAPH            │
│ ingest_pdf(file_bytes, tid)   │        │   State: ChatState{ messages[] }     │
│ 1) temp file write            │        │   Checkpointer: SQLite               │
│ 2) PyPDFLoader.load()         │        └──────────────────────────────────────┘
│ 3) Split into chunks          │                        │
│ 4) Embed chunks               │                        │ START
│ 5) FAISS vector store         │                        ▼
│ 6) retriever per thread       │        ┌──────────────────────────────────────┐
└──────────────────────────────┘        │               chat_node               │
         │                              │ - Adds system instructions            │
         │ stores                        │ - Invokes LLM with tools             │
         ▼                              │ - Produces response OR tool call      │
┌──────────────────────────────┐        └──────────────────────────────────────┘
│ PER-THREAD RETRIEVER STORE    │                        │
│ _THREAD_RETRIEVERS[tid]       │                        │ tools_condition
│ _THREAD_METADATA[tid]         │                        ▼
└──────────────────────────────┘        ┌──────────────────────────────────┐
                                        │              ToolNode             │
                                        │ Executes tool calls:              │
                                        │  • Web Search                     │
                                        │  • Calculator                     │
                                        │  • Stock API                      │
                                        │  • PDF RAG Retriever              │
                                        └──────────────────────────────────┘
                                                │
                                                ▼
                                      ┌──────────────────────────────────────┐
                                      │               chat_node               │
                                      │ LLM synthesizes final response        │
                                      │ using tool outputs                    │
                                      └──────────────────────────────────────┘
                                                │
                                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                               SQLITE CHECKPOINTER                             │
│   - Stores per-thread conversation state                                       │
│   - Enables chat restoration                                                   │
└──────────────────────────────────────────────────────────────────────────────┘
                                                │
                                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     LANGSMITH OBSERVABILITY LAYER                             │
│                                                                              │
│ Automatically traces and logs:                                                │
│  • LLM calls (latency, tokens, prompts, responses)                           │
│  • Tool executions                                                           │
│  • Graph node transitions                                                    │
│  • Errors and retries                                                        │
│  • Execution paths                                                           │
│                                                                              │
│ Enables:                                                                     │
│  • Debugging agent behavior                                                  │
│  • Prompt inspection                                                         │
│  • Performance monitoring                                                    │
│  • Dataset-driven evaluations                                                │
│  • Production reliability                                                    │
└──────────────────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENT / UI                                 │
│  - Streams assistant response                                                 │
│  - Displays tool usage                                                        │
│  - Allows thread switching                                                   │
└──────────────────────────────────────────────────────────────────────────────┘

----------

````markdown
# LangGraph Multi-Utility Chatbot (PDF RAG + Tools + SQLite Memory + LangSmith)

A Streamlit-based multi-utility chatbot built using **LangGraph** + **LangChain**, with:
- **Tool calling** (Web Search, Calculator, Stock API, PDF RAG)
- **Thread-scoped PDF RAG** (each chat thread can index its own PDF using FAISS)
- **SQLite checkpointing** (persistent conversation memory per thread)
- **LangSmith observability** (tracing runs, tool calls, node transitions, errors)

---

## Architecture (High Level)

### What happens in a chat turn?
1. User sends a message from the UI with `thread_id`
2. `chat_node` calls the LLM (tools enabled)
3. If the LLM requests a tool call → LangGraph routes to `ToolNode`
4. Tool runs (search / calculator / stock / rag)
5. Tool output returns to `chat_node`
6. LLM final response is generated
7. Conversation state is checkpointed to SQLite
8. Entire run is traced to LangSmith (when enabled)

### Thread-scoped PDF RAG
- Upload a PDF in the sidebar
- The backend indexes the PDF into a FAISS vector store
- Retriever is stored per `thread_id` so PDFs do not mix across threads

---

## Folder Structure (Recommended)

```text
.
├── langraph_rag_backend.py        # Backend: LangGraph graph + tools + ingestion + SQLite checkpointer
├── streamlit_app.py               # Streamlit UI
├── requirements.txt               # Python dependencies
├── .env                           # Environment variables (NOT committed)
└── chatbot.db                     # SQLite DB created at runtime
````

---

## Requirements

* Python 3.10+ (recommended)
* OpenAI API access (for LLM + embeddings)
* Internet access (optional, for web search + stock tool)
* LangSmith account (optional, for tracing)

---

## Setup

### 1) Create and activate a virtual environment

Windows (PowerShell):

```bash
python -m venv env
env\Scripts\activate
```

macOS / Linux:

```bash
python3 -m venv env
source env/bin/activate
```

### 2) Install dependencies

```bash
pip install -r requirements.txt
```

---

## Environment Variables (.env)

Create a `.env` file in the project root:

```bash
# OpenAI (Required)
OPENAI_API_KEY="your_openai_api_key"

# LangSmith (Optional but recommended)
LANGCHAIN_TRACING_V2="true"
LANGCHAIN_ENDPOINT="https://api.smith.langchain.com"
LANGCHAIN_API_KEY="your_langsmith_key"
LANGCHAIN_PROJECT="chatbot-project2"
```

Important:

* Do not commit `.env` to GitHub.
* If your LangSmith key was shared anywhere publicly, rotate it immediately.

---

## requirements.txt (Suggested)

```text
streamlit
python-dotenv
requests
langchain
langchain-community
langchain-core
langchain-openai
langchain-text-splitters
langgraph
faiss-cpu
pypdf
```

Note:

* On some machines, FAISS install may vary. If `faiss-cpu` fails, try:

  * `pip install faiss-cpu --no-cache-dir`
  * or use a compatible Python version (3.10/3.11 works best)

---

## How to Run

### Start the Streamlit UI

```bash
streamlit run frontend.py
```

Open the URL shown in terminal (usually `http://localhost:8501`).

---

## Step-by-Step Usage

### 1) Start a new chat

* Click **New Chat** in the sidebar
* A new `thread_id` is created and becomes active

### 2) Upload a PDF (optional)

* Use the sidebar uploader: **Upload a PDF for this chat**
* The UI calls `ingest_pdf(file_bytes, thread_id, filename)`
* The backend:

  * reads PDF pages
  * splits into chunks
  * creates embeddings
  * builds FAISS store
  * saves retriever in `_THREAD_RETRIEVERS[thread_id]`

### 3) Ask questions

Examples:

* “Summarize this PDF”
* “What does the document say about X?”
* “Calculate 19.5 * 4.2”
* “Stock price of TSLA”
* “Search web for latest AI agent frameworks”

### 4) Observe tool usage (UI)

* When the LLM uses a tool, the UI shows a live status box:

  * “Using `tool_name` …”
  * then “Tool finished”

### 5) Switch between past threads

* Sidebar shows **Past conversations**
* Clicking a thread loads conversation state from SQLite via:

  * `chatbot.get_state(config={"configurable": {"thread_id": ...}})`

---

## Backend Details

### Tools available

* Web Search: `DuckDuckGoSearchRun`
* Calculator: `calculator(first_num, second_num, operation)`
* Stock API: `get_stock_price(symbol)` (Alpha Vantage)
* PDF RAG: `rag_tool(query, thread_id)` (thread-scoped retriever)

### Persistence

* Uses LangGraph `SqliteSaver` with `chatbot.db`
* Stores conversation checkpoints per `thread_id`

### Observability (LangSmith)

When LangSmith env vars are enabled, LangChain/LangGraph automatically traces:

* LLM calls (latency, tokens, prompts, responses)
* Tool executions
* Graph node transitions
* Errors and retries
* Execution paths

You can view runs inside your LangSmith project:

* Project: `LANGCHAIN_PROJECT`

---

## Notes / Production Improvements

* Move AlphaVantage key to `.env` instead of hardcoding
* Persist vector stores (FAISS) if you need restart durability
* Add caching to avoid re-indexing same PDFs
* Add auth + rate limiting if exposed publicly
* Add eval + regression checks (datasets) in LangSmith

---

## Author
Karan Kadam

