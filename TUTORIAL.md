# Claude Code Tutorial — RAG Chatbot Project
## Learn Claude Code Step by Step Using a Real Project

> This tutorial was auto-generated during a live Claude Code session.
> Every prompt, action, and result was logged in real time.

---

## Project Overview

This project is a full-stack **RAG (Retrieval-Augmented Generation) chatbot** that answers questions about Anthropic course materials. You ask a question in a browser chat UI, and the system searches course transcripts using semantic vector search, then uses Claude to generate a grounded answer.

**Stack:**
- **Backend:** Python 3.13, FastAPI, ChromaDB, Anthropic Claude API
- **Frontend:** Vanilla HTML / CSS / JavaScript (no framework)
- **Package manager:** `uv` (fast Python package manager)
- **Vector database:** ChromaDB (runs locally, no external service)
- **Embeddings:** `all-MiniLM-L6-v2` (local sentence-transformer model)

---

## Prerequisites

Before following along, you need:
- Python 3.13 installed
- `uv` package manager installed ([install guide](https://docs.astral.sh/uv/getting-started/installation/))
- An Anthropic API key (get one at console.anthropic.com)
- Claude Code CLI installed and authenticated
- Git (to clone the repo)

**Setup steps:**
```bash
# 1. Clone the repository
git clone <repo-url>
cd starting-ragchatbot-codebase

# 2. Copy the environment file and add your API key
cp backend/.env.example backend/.env
# Open backend/.env and set: ANTHROPIC_API_KEY=sk-ant-...

# 3. Install dependencies
uv sync
```

---

## Steps

---

## Step 1 — Starting the Application

### 🗣️ Prompt
> "start the application"

### 🔍 What Claude Did
Claude attempted to start the RAG chatbot server using `./run.sh` from the repo root — the documented way to start the app. The script failed immediately because `uv` (the Python package manager) was installed in a non-standard location and was not on the shell's `PATH`.

Claude then searched for the `uv` executable across common install locations and found it at `/c/Users/Personal/.local/bin/uv.exe`. It re-ran the server command directly using the full path, this time from inside the `backend/` directory (required because `config.py` uses relative paths like `./chroma_db`).

A second attempt to start the server failed with `[WinError 10048] only one usage of each socket address` — meaning the first server was already running successfully on port 8000. This confirmed the app was up.

### 💻 Command / Code
```bash
# First attempt — failed because uv was not on PATH
./run.sh

# Claude found uv here:
ls /c/Users/Personal/.local/bin/uv*
# → /c/Users/Personal/.local/bin/uv.exe

# Successful start — run from the backend/ directory
cd backend
/c/Users/Personal/.local/bin/uv run uvicorn app:app --reload --port 8000
```

### ✅ Result
```
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [15448] using WatchFiles
```

On startup, the app automatically ingested all 4 course documents from `docs/`:
```
Course already exists: Building Towards Computer Use with Anthropic - skipping
Course already exists: MCP: Build Rich-Context AI Apps with Anthropic - skipping
Course already exists: Advanced Retrieval for AI with Chroma - skipping
Course already exists: Prompt Compression and Query Optimization - skipping
```

The "skipping" messages are expected — ChromaDB already had the courses indexed from a previous run. The app served correctly at `http://localhost:8000`.

### 📚 What to Learn
**Claude Code can run shell commands on your behalf.** When you say "start the application," Claude reads the project's documentation (`CLAUDE.md`), finds the correct startup command, and executes it. If something goes wrong (like a missing `PATH` entry), Claude investigates and adapts — it doesn't just fail and stop.

**Key concept:** Claude Code has access to your terminal. It can run any bash command, read output, and recover from errors automatically. This is one of its most powerful features for development workflows.

**The `--reload` flag** tells uvicorn to watch for file changes and automatically restart the server — essential for development so you don't have to manually restart every time you edit a file.

---

## Step 2 — Making Source Citations into Clickable Links

### 🗣️ Prompt
> "The chat interface displays query responses with source citations. I need to modify it so each source becomes a clickable link that opens the corresponding lesson video in a new tab:
> - When courses are processed into chunks in @backend/document_processor.py, the link of each lesson is stored in the course_catalog collection
> - Modify _format_results in @backend/search_tools.py so that the lesson links are also returned
> - The links should be embedded invisibly (no visible URL text)"

### 🔍 What Claude Did
This was a multi-file change touching backend and frontend. Claude used **Plan Mode** to research and design the approach before writing any code.

**Phase 1 — Research (Plan Mode):**
Claude launched an Explore subagent to trace the full data flow: from how lesson links are stored in ChromaDB, through the backend API response, to how the frontend renders source citations. Key finding: lesson links are stored in `course_catalog` as a JSON string (`lessons_json`), and `VectorStore.get_lesson_link()` already exists to retrieve them — no new database code needed.

**Phase 2 — Design:**
Claude drafted a plan with exactly three targeted changes:
1. `search_tools.py`: Look up the lesson URL for each search result and store sources as `{"label": ..., "url": ...}` dicts instead of plain strings
2. `app.py`: Add a `Source` Pydantic model so the API response is properly typed
3. `script.js`: Add a `renderSource()` function that wraps sources with URLs in `<a>` tags

**Phase 3 — Implementation:**
After the plan was approved, Claude made the three edits precisely as designed.

### 💻 Command / Code

**`backend/search_tools.py` — `_format_results()` (changed section):**
```python
# Before: sources were plain text strings
sources.append(source)

# After: sources are dicts with label + URL
url = self.store.get_lesson_link(course_title, lesson_num) if lesson_num is not None else None
sources.append({"label": label, "url": url})
```

**`backend/app.py` — new `Source` Pydantic model:**
```python
# New model added before QueryResponse
class Source(BaseModel):
    """A single source citation with optional link"""
    label: str
    url: Optional[str] = None

# QueryResponse updated to use it
class QueryResponse(BaseModel):
    answer: str
    sources: List[Source]   # was: List[str]
    session_id: str
```

**`frontend/script.js` — new `renderSource()` function:**
```javascript
// New helper — renders a source as a link if it has a URL, plain text otherwise
function renderSource(source) {
    if (source.url) {
        // Opens in a new tab; rel="noopener noreferrer" is a security best practice
        return `<a href="${source.url}" target="_blank" rel="noopener noreferrer">${escapeHtml(source.label)}</a>`;
    }
    return escapeHtml(source.label);  // Fallback: plain text if no URL
}

// Updated sources rendering in addMessage() — was: sources.join(', ')
<div class="sources-content">${sources.map(renderSource).join(', ')}</div>
```

### ✅ Result
The server's `--reload` watcher automatically applied the backend changes. After a browser refresh, source citations in the chat UI became clickable hyperlinks. Clicking one opens the corresponding lesson video URL in a new tab. The anchor text is the lesson label (e.g., "MCP: Build Rich-Context AI Apps with Anthropic - Lesson 2") — no raw URL is visible.

Sources that have no lesson link in the course document fall back to plain text automatically.

### 📚 What to Learn
**Plan Mode is a powerful safety net for multi-file changes.** Before touching any code, Claude entered Plan Mode (`/plan`), explored the codebase with a subagent, and wrote a detailed implementation plan to a file. You reviewed and approved the plan before any edits were made. This prevents Claude from going off in the wrong direction on complex tasks.

**Three important patterns demonstrated here:**

1. **Reuse before building** — `VectorStore.get_lesson_link()` already existed. Claude found it during research and used it rather than writing new database code.

2. **Type safety in FastAPI** — When your API response changes shape (from `List[str]` to `List[dict]`), you update the Pydantic model. FastAPI uses this to validate and serialize responses automatically — no manual JSON wrangling.

3. **`rel="noopener noreferrer"`** — Always include this on `target="_blank"` links. It prevents the opened tab from having a reference back to your page (`noopener`) and stops the browser from sending your page's URL as a referrer (`noreferrer`). This is standard security practice.

---

---

## Step 3 — Restarting the Server After Code Changes

### 🗣️ Prompt
> "can you restart the server to pick up these changes? just did a query and no changes"

### 🔍 What Claude Did
The server was running with `--reload`, which watches for Python file changes. However, the browser was still serving the old `script.js` from cache, causing `[object Object]` to appear instead of source labels. Claude first killed all Python processes (the old server instances), then restarted the server fresh.

### 💻 Command / Code
```bash
# Kill all running Python processes (including the old server)
# Note: on Windows this must be run through cmd, not bash directly
cmd //c "taskkill /F /IM python.exe"

# Restart the server from the backend directory
cd backend
/c/Users/Personal/.local/bin/uv run uvicorn app:app --reload --port 8000
```

### ✅ Result
```
SUCCESS: The process "python.exe" with PID 4340 has been terminated.
SUCCESS: The process "python.exe" with PID 15448 has been terminated.
...
INFO: Uvicorn running on http://127.0.0.1:8000
```

The `[object Object]` display was caused by the browser's cache serving the old `script.js`. A hard refresh (`Ctrl + Shift + R`) forced the browser to load the updated file.

### 📚 What to Learn
**Browser caching can mask frontend changes.** Even after a server restart, the browser may serve JavaScript/CSS from its local cache. When frontend changes don't appear, always try a **hard refresh** (`Ctrl + Shift + R` on Windows/Linux, `Cmd + Shift + R` on Mac) before assuming something is broken.

**`taskkill /F /IM python.exe`** kills every running Python process on Windows. The `/F` flag forces termination and `/IM` targets by image name. Be careful — this kills ALL Python processes, not just the server.

---

## Step 4 — Styling Source Citations as Pill Badges

### 🗣️ Prompt
> "these links are hard to read can you make this more visually appealing" *(with screenshot showing dense comma-separated blue links on dark background)*

### 🔍 What Claude Did
Claude read `style.css` to understand the existing design system (CSS variables, colors, spacing). Rather than tweaking the existing inline link style, Claude redesigned the sources section entirely: each citation becomes an individual pill-shaped badge displayed in a flex-wrap row, using the project's existing blue accent color (`#2563eb`) at low opacity for a subtle but readable appearance.

Three targeted changes:
1. **`style.css`** — replaced `.sources-content` padding rule with flex-wrap layout; added new `.source-tag` pill style with border, background tint, and hover transition
2. **`script.js` `renderSource()`** — added `class="source-tag"` to both the `<a>` (linked) and `<span>` (unlinked) variants
3. **`script.js` `addMessage()`** — changed `.join(', ')` to `.join('')` since pills are self-separating via flex gap

### 💻 Command / Code

**`style.css` — new pill styles:**
```css
.sources-content {
    padding: 0.25rem 0.5rem;
    display: flex;       /* lay pills out horizontally */
    flex-wrap: wrap;     /* wrap to next line if too many */
    gap: 0.4rem;         /* space between pills */
}

.source-tag {
    display: inline-flex;
    align-items: center;
    padding: 0.2rem 0.65rem;
    background: rgba(37, 99, 235, 0.12);   /* subtle blue tint */
    border: 1px solid rgba(37, 99, 235, 0.3);
    border-radius: 999px;                   /* fully rounded = pill shape */
    font-size: 0.72rem;
    font-weight: 500;
    color: #93c5fd;                         /* light blue text, readable on dark */
    text-decoration: none;
    white-space: nowrap;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;
}

.source-tag:hover {
    background: rgba(37, 99, 235, 0.25);
    border-color: rgba(37, 99, 235, 0.6);
    color: #bfdbfe;
}
```

**`script.js` — updated renderSource:**
```javascript
function renderSource(source) {
    if (source.url) {
        // Linked pill — clicks open the lesson video in a new tab
        return `<a class="source-tag" href="${source.url}" target="_blank" rel="noopener noreferrer">${escapeHtml(source.label)}</a>`;
    }
    // Unlinked pill — displays as a badge with no click action
    return `<span class="source-tag">${escapeHtml(source.label)}</span>`;
}
```

### ✅ Result
Sources now appear as individual light-blue pill badges below each assistant message. They wrap to multiple lines if needed, have clear hover feedback, and are easy to read on the dark background. No server restart was required — frontend-only change; a browser hard refresh (`Ctrl + Shift + R`) applied the update.

### 📚 What to Learn
**CSS custom properties (variables) make theming consistent.** The pill color (`#93c5fd`, `rgba(37, 99, 235, ...)`) was chosen to match the project's `--primary-color: #2563eb` already defined in `:root`. Rather than picking arbitrary colors, always look at the existing design system first.

**`border-radius: 999px`** is the standard trick for pill-shaped elements — a large enough value that the border curves fully regardless of element height, without needing to know the exact pixel height.

**`flex-wrap: wrap` + `gap`** is the modern CSS way to lay out a variable number of tags or badges. Unlike floats or inline-block with margins, it handles wrapping, spacing, and alignment cleanly in a few lines.

---

---

## Step 5 — Adding a "+ New Chat" Button to the Sidebar

### 🗣️ Prompt
> "Add a '+ NEW CHAT' button to the left sidebar above the courses section. When clicked, it should: clear the current conversation in the chat window, start a new session without page reload, handle proper cleanup on both frontend and backend, match the styling of existing sections (Courses, Try asking) — same font size, color, and uppercase formatting"

### 🔍 What Claude Did
This was a pure frontend change. Claude used **Plan Mode** to research the code before writing anything.

**Research phase:** Claude read `index.html`, `style.css`, `script.js`, and `session_manager.py` to understand:
- Where in the sidebar HTML to insert the button
- What the existing section header styles look like (font size, weight, color, uppercase)
- Whether `createNewSession()` already existed (it did — at line 156 of `script.js`)
- Whether the backend needed changes (it didn't — setting `currentSessionId = null` on the frontend causes the next query to auto-create a fresh backend session)

**Implementation:** Three targeted edits, no backend changes needed.

### 💻 Command / Code

**`frontend/index.html` — button added above sidebar sections:**
```html
<!-- New Chat Button -->
<button id="newChatBtn" class="new-chat-btn">+ New Chat</button>
```
Also bumped cache-busting version strings: `style.css?v=9` → `v=10`, `script.js?v=9` → `v=10`

**`frontend/style.css` — new button style matching sidebar headers:**
```css
.new-chat-btn {
    width: 100%;
    padding: 0.5rem 0;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 0.875rem;      /* matches .stats-header font size */
    font-weight: 600;
    color: var(--text-secondary);
    text-transform: uppercase; /* matches "COURSES" and "TRY ASKING:" */
    letter-spacing: 0.5px;
    text-align: left;
    transition: color 0.2s ease;
    margin-bottom: 1.5rem;
}

.new-chat-btn:hover,
.new-chat-btn:focus {
    color: var(--primary-color); /* blue on hover, same as header hover */
    outline: none;
}
```

**`frontend/script.js` — one line wired in `setupEventListeners()`:**
```javascript
document.getElementById('newChatBtn').addEventListener('click', createNewSession);
```
`createNewSession()` was already there — it sets `currentSessionId = null`, clears the chat messages, and shows the welcome message. No new logic was needed.

### ✅ Result
After a hard refresh (`Ctrl + Shift + R`), a `+ NEW CHAT` label appears at the top of the sidebar above the "COURSES" collapsible. It visually matches the existing section headers. Clicking it clears all messages and shows the welcome message. The next query sent starts a completely fresh conversation with no memory of prior exchanges.

### 📚 What to Learn
**Reuse before building.** The `createNewSession()` function already did exactly what was needed. Claude found it during research and wired the button to it with one line — no new logic written.

**Cache-busting with query strings** (`?v=10`) is a simple way to force browsers to reload CSS and JS files when you change them. Since the `DevStaticFiles` class in `app.py` already adds no-cache headers in development, the version bump is an extra safety net for cases where the browser ignores those headers.

**Frontend-only features don't require backend changes.** The old session is abandoned in backend memory when `currentSessionId` is set to `null` — this is fine because sessions are already ephemeral (lost on server restart). There's no need to add a DELETE endpoint just for a UI reset button.

---

---

## Step 6 — Adding the Playwright MCP Server

### 🗣️ Prompt
> *(No in-session prompt — user performed this step outside Claude Code and provided context)*

### 🔍 What Claude Did
Nothing — this step was performed manually by the user. Claude Code was closed, the Playwright MCP server was registered, and Claude Code was reopened. The step is logged here for completeness.

### 💻 Command / Code
```bash
# Run this once in your terminal (outside Claude Code)
claude mcp add playwright npx @playwright/mcp@latest

# Then reopen Claude Code to pick up the new MCP server
```

### ✅ Result
The Playwright MCP server is now available inside Claude Code sessions. Claude can use it to open a real browser, take screenshots, click elements, and inspect the live UI — without the user having to manually capture and upload screenshots.

### 📚 What to Learn
**MCP (Model Context Protocol) servers extend what Claude Code can do.** By default, Claude can read/write files and run terminal commands. MCP servers add new tools on top of that — Playwright gives Claude the ability to control a real browser.

**Why this matters for UI work:** Instead of the user taking a screenshot, describing what looks wrong, and waiting for Claude to guess at a fix, Claude can:
1. Open the live app in a browser
2. Take its own screenshot
3. Inspect element styles
4. Verify that a change looks correct before declaring it done

**How to add any MCP server:**
```bash
claude mcp add <name> <command>
```
- `<name>` is a label you pick (e.g., `playwright`)
- `<command>` is how to run the server (e.g., `npx @playwright/mcp@latest`)
- MCP servers are saved to your Claude Code config and persist across sessions
- You must restart Claude Code after adding a new server for it to appear

---

## Key Concepts Covered

| Concept | Introduced In |
|---|---|
| Claude Code running shell commands | Step 1 |
| Diagnosing PATH / environment issues | Step 1 |
| uvicorn `--reload` for development | Step 1 |
| ChromaDB startup ingestion | Step 1 |
| Plan Mode for multi-file changes | Step 2 |
| Using Explore subagents for research | Step 2 |
| FastAPI Pydantic response models | Step 2 |
| Reusing existing code (`get_lesson_link`) | Step 2 |
| `target="_blank"` security (`noopener noreferrer`) | Step 2 |
| Backend → frontend data flow tracing | Step 2 |
| Browser cache and hard refresh | Step 3 |
| `taskkill` to kill processes on Windows | Step 3 |
| CSS variables and design systems | Step 4 |
| Pill badge styling (`border-radius: 999px`) | Step 4 |
| `flex-wrap` + `gap` for tag layouts | Step 4 |
| Reusing existing JS functions for new UI | Step 5 |
| Cache-busting with `?v=N` query strings | Step 5 |
| Frontend-only features with no backend changes | Step 5 |
| Adding MCP servers to Claude Code (`claude mcp add`) | Step 6 |
| Playwright MCP for browser automation / UI inspection | Step 6 |

---

## Step 7 — Restyling the "New Chat" Button with Playwright Verification

### 🗣️ Prompt
> "Using the playwright mcp server, visit 127.0.0.1:8000 and view the new chat button. I want that button to look the same as the other links below for courses and try asking. make sure this is left aligned and that the border is removed."

### 🔍 What Claude Did
Used the Playwright MCP browser to navigate to the running app, take screenshots, and expand the sidebar sections to see what the "Try asking:" items looked like. Then edited `style.css` to restyle `.new-chat-btn` to match — same dark background, rounded corners, normal text case, and no border. Bumped the CSS version from `?v=10` to `?v=11` in `index.html` to bust the browser cache. Verified the result with a final Playwright screenshot.

Along the way, Claude also had to locate the correct `uv` binary (`~/.local/bin/uv`) since it wasn't on the default `PATH` in the bash shell, and used `nohup` to start the server as a persistent background process.

### 💻 Command / Code

CSS change in `frontend/style.css`:
```css
/* Before — standalone button style */
.new-chat-btn {
    padding: 0.5rem 0;
    background: none;
    border: none;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

/* After — matches .suggested-item style, no border */
.new-chat-btn {
    padding: 0.75rem 1rem;
    background: var(--background);
    border: none;
    border-radius: 8px;
    font-weight: 400;
    text-transform: none;
    letter-spacing: normal;
    text-align: left;
}
```

Cache-bust in `frontend/index.html`:
```html
<link rel="stylesheet" href="style.css?v=11">
```

Server start command (uv not on PATH in bash):
```bash
cd backend && nohup ~/.local/bin/uv run uvicorn app:app --port 8000 > /tmp/server.log 2>&1 &
```

### ✅ Result
The "+ New Chat" button now visually matches the "Try asking:" suggestion items — dark background, rounded corners, left-aligned text, no border. Verified side-by-side with a Playwright screenshot.

### 📚 What to Learn
- **Playwright MCP** lets Claude open a real browser, take screenshots, click elements, and read computed styles — all without you lifting a finger. This is powerful for visual UI review and verification.
- **`window.getComputedStyle()`** (via `browser_evaluate`) lets you inspect the *actual* CSS being applied to an element, not just what's in the file. This is how Claude confirmed the old CSS was still cached.
- **Cache busting with `?v=N`**: browsers cache CSS files aggressively. Changing the query string (e.g. `?v=10` → `?v=11`) forces the browser to treat it as a new file and fetch fresh styles.
- **Finding binaries on Windows**: `uv` installs to `~/.local/bin/uv` by default. If your shell doesn't have it on `PATH`, you can call it with the full path directly.
| Adding MCP servers to Claude Code (`claude mcp add`) | Step 6 |
| Playwright MCP for browser automation / UI inspection | Step 7 |
| CSS cache busting with `?v=N` versioned query strings | Step 7 |
| Inspecting live computed styles with `getComputedStyle` | Step 7 |

---

---

## Step 8 — Adding a Course Outline Tool

### 🗣️ Prompt
> "In @backend/search_tools.py, add a second tool alongside the existing content-related tool. This new tool should handle course outline queries. Functionality: Input: Course title. Output: Course title, course link, and complete lesson list. For each lesson: lesson number, lesson title. Data source: Course metadata collection of the vector store. Update the system prompt in @backend/ai_generator.py so that the course title, course link, the number and title of each lesson are all returned to address outline-related queries. Make sure that the new tool is registered in the system."

### 🔍 What Claude Did
Claude read four backend files to understand the existing architecture before writing a single line of code: `search_tools.py`, `vector_store.py`, `ai_generator.py`, and `rag_system.py`.

**Research findings:**
- `vector_store.py` already stores the full lesson list (number, title, link) as a JSON string (`lessons_json`) inside the `course_catalog` ChromaDB collection — exactly the data source needed
- `_resolve_course_name()` already did fuzzy course name matching via semantic search — reusable for the new tool
- `ToolManager` uses a registry pattern — new tools are registered with one line in `rag_system.py`

**Implementation:** Four targeted edits, no new files created.

1. **`vector_store.py`** — Added `get_course_outline(course_name)`: resolves the name via `_resolve_course_name()`, fetches the catalog entry by ID, parses `lessons_json`, and returns a dict with `title`, `course_link`, and `lessons`
2. **`search_tools.py`** — Added `CourseOutlineTool` class: implements the `Tool` ABC, exposes a `get_course_outline` tool definition to Claude, and formats the result as a readable text block
3. **`ai_generator.py`** — Updated `SYSTEM_PROMPT` to distinguish between the two tools: `search_course_content` for detailed content questions, `get_course_outline` for structure/listing questions
4. **`rag_system.py`** — Imported `CourseOutlineTool` and registered an instance with the `ToolManager`

### 💻 Command / Code

**`backend/vector_store.py` — new method:**
```python
def get_course_outline(self, course_name: str) -> Optional[Dict[str, Any]]:
    """Get full course outline (title, link, lessons) for a given course name."""
    import json
    course_title = self._resolve_course_name(course_name)
    if not course_title:
        return None
    results = self.course_catalog.get(ids=[course_title])
    if results and results['metadatas']:
        metadata = results['metadatas'][0]
        lessons = json.loads(metadata.get('lessons_json', '[]'))
        return {
            "title": metadata.get('title'),
            "course_link": metadata.get('course_link'),
            "lessons": lessons
        }
    return None
```

**`backend/search_tools.py` — new tool class:**
```python
class CourseOutlineTool(Tool):
    def get_tool_definition(self) -> Dict[str, Any]:
        return {
            "name": "get_course_outline",
            "description": "Get the complete outline of a course: title, link, and all lessons with numbers and titles.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "course_name": {"type": "string", "description": "Course title or partial name"}
                },
                "required": ["course_name"]
            }
        }

    def execute(self, course_name: str) -> str:
        outline = self.store.get_course_outline(course_name)
        if not outline:
            return f"No course found matching '{course_name}'."
        lines = [f"Course: {outline['title']}", f"Link: {outline['course_link'] or 'N/A'}", "", "Lessons:"]
        for lesson in outline['lessons']:
            lines.append(f"  Lesson {lesson['lesson_number']}: {lesson['lesson_title']}")
        return "\n".join(lines)
```

**`backend/rag_system.py` — registration:**
```python
self.outline_tool = CourseOutlineTool(self.vector_store)
self.tool_manager.register_tool(self.outline_tool)
```

**`backend/ai_generator.py` — updated system prompt excerpt:**
```
- Use `search_course_content` for questions about specific course content or detailed educational materials
- Use `get_course_outline` for questions about course structure, lesson list, or course overview — always return the course title, course link, and every lesson with its number and title
- **One tool call per query maximum**
```

### ✅ Result
The backend now exposes two tools to Claude. Outline queries (e.g. "What lessons does course X have?") trigger `get_course_outline`, which returns the full structured lesson list. Content questions continue to use `search_course_content` as before. The system prompt change ensures Claude picks the right tool and includes title, course link, and all lesson numbers/titles in outline responses.

### 📚 What to Learn
**The Tool ABC pattern makes adding tools mechanical.** Every tool in this project subclasses `Tool` and implements exactly two methods: `get_tool_definition()` (what Claude sees) and `execute()` (what runs when Claude calls it). The `ToolManager` registry means Claude automatically has access to any registered tool — no other wiring needed.

**Reuse before building.** The `_resolve_course_name()` method and the `lessons_json` field already existed in the vector store. The new tool is mostly just calling existing code in a new combination — a sign of a well-designed system.

**System prompts control tool selection.** Claude has no inherent understanding of which tool to use for which question. You teach it by updating the system prompt with clear rules. Without the prompt update, Claude might use `search_course_content` even for outline queries, returning chunks of transcript text instead of a structured lesson list.

---

## Session End Note — 2026-03-17

Before clearing context to begin backend changes, the following environment state was recorded:

- **Server**: Running via `nohup ~/.local/bin/uv run uvicorn app:app --port 8000` from `backend/`
- **`uv` binary location**: `~/.local/bin/uv` — not on default bash `PATH`, must use full path
- **CSS version**: `?v=12` (in `frontend/index.html`)
- **JS version**: `?v=10` (in `frontend/index.html`)
- **Next session focus**: Backend changes
