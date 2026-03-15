# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## What This Project Is

A full-stack RAG (Retrieval-Augmented Generation) chatbot that answers questions about course materials. Users interact via a browser chat UI; the backend uses semantic search over ChromaDB and Anthropic's Claude API (with tool use) to generate grounded answers from course transcripts.

---

## Commands

```bash
# Install dependencies (Python 3.13 required)
uv sync

# Run the server — must be run from repo root
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

- App UI: `http://localhost:8000`
- API docs (Swagger): `http://localhost:8000/docs`
- Requires `ANTHROPIC_API_KEY` set in `.env` (copy from `.env.example`)

`main.py` at the repo root is a placeholder stub — it is not used to run the application.

---

## Project Structure

```
├── backend/
│   ├── app.py               # FastAPI app, routes, startup ingestion, static file serving
│   ├── rag_system.py        # Main orchestrator — wires all components together
│   ├── ai_generator.py      # Anthropic API calls, tool-use loop
│   ├── search_tools.py      # Tool abstraction layer (Tool, CourseSearchTool, ToolManager)
│   ├── vector_store.py      # ChromaDB interface (two collections, semantic search)
│   ├── document_processor.py# File parsing, lesson extraction, text chunking
│   ├── session_manager.py   # In-memory conversation history per session
│   ├── models.py            # Pydantic models: Course, Lesson, CourseChunk
│   ├── config.py            # All tunable parameters (loaded from .env)
│   └── chroma_db/           # Persisted ChromaDB data (gitignored)
├── frontend/
│   ├── index.html           # Single-page chat UI with sidebar
│   ├── script.js            # All frontend logic (fetch, session, rendering)
│   └── style.css            # Dark theme, responsive layout (CSS variables)
├── docs/
│   ├── course1_script.txt   # "Building Towards Computer Use with Anthropic"
│   ├── course2_script.txt   # Course transcript 2
│   ├── course3_script.txt   # Course transcript 3
│   └── course4_script.txt   # Course transcript 4
├── .env                     # Local secrets (gitignored)
├── .env.example             # Template — update this when adding new env vars
├── pyproject.toml           # Dependencies managed by uv
└── run.sh                   # Startup script
```

---

## Architecture

### High-Level Flow

The FastAPI backend serves both the REST API (`/api/*`) and the static frontend files (`/`). There is no separate frontend server — everything runs on port 8000.

```
Browser → POST /api/query
       → RAGSystem.query()
       → AIGenerator (1st Claude call, tool_choice=auto)
         ├─ no tool needed → return answer directly
         └─ tool_use → CourseSearchTool.execute()
                          → VectorStore.search() → ChromaDB
                       → AIGenerator (2nd Claude call, synthesize results)
       → return { answer, sources, session_id }
```

### Component Responsibilities

**`app.py`**
- Initializes `RAGSystem` once on startup
- On startup event, loads all `.txt/.pdf/.docx` files from `../docs/` into ChromaDB (skips already-indexed courses by title)
- `POST /api/query` — resolves/creates session, delegates to `rag_system.query()`
- `GET /api/courses` — returns course count and titles from vector store
- Mounts `../frontend` as static files at `/` with `html=True` (serves `index.html` for all unmatched routes)
- `DevStaticFiles` subclass adds no-cache headers (development convenience, not used for the main mount)

**`rag_system.py`**
- Single `RAGSystem` class that holds all component instances
- `query()` method: wraps user query in prompt, fetches session history, calls AI generator with tools, collects sources from tool manager, saves exchange to session
- `add_course_folder()`: idempotent — checks existing titles before ingesting, supports `clear_existing=True` for full rebuild
- `get_course_analytics()`: delegates to vector store for count + title list

**`ai_generator.py`**
- `generate_response()`: builds API params, appends conversation history to system prompt, calls Claude with `tool_choice: auto`
- `_handle_tool_execution()`: when `stop_reason == "tool_use"`, executes all tool calls via `ToolManager`, appends `tool_result` blocks to messages, makes a second Claude call **without tools** to get the final synthesized answer
- The system prompt enforces: one search per query max, no meta-commentary ("based on search results"), direct answers only
- Model: `claude-sonnet-4-20250514`, temperature: 0, max_tokens: 800

**`search_tools.py`**
- `Tool` (ABC): interface with `get_tool_definition()` and `execute()`
- `CourseSearchTool`: implements `search_course_content` tool — accepts `query` (required), `course_name` (optional, fuzzy match), `lesson_number` (optional int). Calls `VectorStore.search()`, formats results with `[Course - Lesson N]` headers, stores `last_sources` for UI display
- `ToolManager`: registry pattern — `register_tool()`, `execute_tool()`, `get_tool_definitions()`, `get_last_sources()`, `reset_sources()`

**`vector_store.py`**
- Two ChromaDB persistent collections:
  - `course_catalog`: one document per course (title text + metadata). Used for fuzzy course name resolution via semantic search. Course `title` is the ChromaDB document ID.
  - `course_content`: one document per chunk. Metadata includes `course_title`, `lesson_number`, `chunk_index`. IDs are `{course_title}_{chunk_index}`.
- Embedding model: `all-MiniLM-L6-v2` (via `sentence-transformers`, runs locally)
- `search()`: resolves `course_name` → exact title via `_resolve_course_name()`, builds ChromaDB `where` filter, queries `course_content`
- `_build_filter()`: handles three cases — course only, lesson only, or `$and` both
- Lesson links and metadata are stored as JSON string (`lessons_json`) in ChromaDB metadata (ChromaDB doesn't support nested objects)

**`document_processor.py`**
- `process_course_document()`: parses the required 3-line header (Course Title, Course Link, Course Instructor), then extracts lessons via `Lesson N: Title` regex markers, optionally followed by `Lesson Link: url`
- `chunk_text()`: sentence-aware chunking — splits on sentence boundaries (regex handles abbreviations), builds chunks up to `CHUNK_SIZE` (800 chars) with `CHUNK_OVERLAP` (100 chars) between consecutive chunks
- First chunk of each lesson is prefixed with `"Lesson N content: "` for retrieval context
- Fallback: if no lesson markers found, entire document content is chunked as one block

**`session_manager.py`**
- Pure in-memory dict keyed by session ID (`session_1`, `session_2`, etc.)
- `MAX_HISTORY=2` — keeps last 2 exchanges (4 messages: 2 user + 2 assistant)
- Conversation history is formatted as plain text and injected into the system prompt (not as separate message objects)
- Sessions are lost on server restart — there is no persistence

**`models.py`**
- `Course(title, course_link, instructor, lessons[])` — `title` is the unique identifier used as ChromaDB ID
- `Lesson(lesson_number, title, lesson_link)` — lesson_number is an int
- `CourseChunk(content, course_title, lesson_number, chunk_index)` — the unit stored in ChromaDB

**`config.py`**
- All parameters in one `Config` dataclass. Change here to affect the whole system:

| Parameter | Default | Effect |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model used for generation |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Local sentence-transformer for embeddings |
| `CHUNK_SIZE` | 800 | Max characters per chunk |
| `CHUNK_OVERLAP` | 100 | Overlap between consecutive chunks |
| `MAX_RESULTS` | 5 | Max ChromaDB results returned per search |
| `MAX_HISTORY` | 2 | Number of conversation exchanges retained |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB persistence directory (relative to `backend/`) |

### Frontend (`frontend/`)

- Single-page app with no framework — vanilla HTML/CSS/JS
- **Layout**: fixed 320px left sidebar (collapsible course stats + suggested questions) + main chat area (max-width 800px, centered)
- **Dark theme** via CSS custom properties defined in `:root` — all colors reference variables, making theme changes a single-file edit
- `script.js` manages: session ID state, `sendMessage()` → `POST /api/query`, `loadCourseStats()` → `GET /api/courses`, loading spinner, markdown rendering via CDN `marked.js`, collapsible sources block
- User messages are HTML-escaped; assistant messages are parsed as Markdown via `marked.parse()`
- Responsive breakpoints: collapses to vertical stack at ≤768px (sidebar moves below chat, max-height 40vh); sidebar narrows to 280px at ≤1024px

---

## Course Document Format

New course files added to `docs/` must follow this exact format:

```
Course Title: [Full Course Title]
Course Link: [https://...]
Course Instructor: [Instructor Name]

Lesson 0: Introduction
Lesson Link: [https://...]
[lesson transcript content...]

Lesson 1: Topic Name
Lesson Link: [https://...]
[lesson transcript content...]
```

- The parser uses `course.title` as the unique ID — duplicate titles will be skipped on re-ingestion
- Supported file extensions: `.txt`, `.pdf`, `.docx`
- `Lesson Link:` line directly after a `Lesson N:` header is consumed as metadata, not content

---

## Extending the System

### Adding a New Search Tool

1. Subclass `Tool` in `search_tools.py`:
```python
class MyTool(Tool):
    def get_tool_definition(self) -> Dict[str, Any]:
        return {
            "name": "my_tool_name",
            "description": "...",
            "input_schema": { "type": "object", "properties": {...}, "required": [...] }
        }

    def execute(self, **kwargs) -> str:
        # return a string result
        ...
```
2. Register in `RAGSystem.__init__()`:
```python
self.my_tool = MyTool(...)
self.tool_manager.register_tool(self.my_tool)
```
Claude will automatically have access to it on the next query.

### Adding a New API Endpoint

Add a route to `app.py`. The `rag_system` instance is module-level and accessible within any route handler.

### Re-indexing Documents

To force a full re-index (e.g. after changing chunk size):
```python
rag_system.add_course_folder("../docs", clear_existing=True)
```
Or delete `backend/chroma_db/` and restart the server — the startup event will re-ingest everything.

---

## Key Constraints and Gotchas

- **Sessions are in-memory only** — lost on every server restart. There is no database-backed session persistence.
- **One tool call per query** — enforced by the system prompt, not code. Claude is instructed not to search more than once.
- **Second Claude call has no tools** — `_handle_tool_execution()` deliberately omits the `tools` param to prevent infinite tool loops.
- **ChromaDB IDs must be unique strings** — course title is used directly as ID; spaces in titles are preserved in `course_catalog` but replaced with `_` in `course_content` chunk IDs.
- **Embedding model runs locally** — `all-MiniLM-L6-v2` is downloaded by `sentence-transformers` on first run (~90MB). No external embedding API is used.
- **`chroma_db/` is gitignored** — each developer/deployment starts with an empty vector store and ingests on first startup.
- **`backend/` is the working directory** — `CHROMA_PATH = "./chroma_db"` and `docs_path = "../docs"` in startup event are both relative to `backend/`, not repo root. Run the server from `backend/` or via `run.sh` which handles this.
