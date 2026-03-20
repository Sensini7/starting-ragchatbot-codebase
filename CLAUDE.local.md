# CLAUDE.local.md — RAG Chatbot Project (Local Instructions)
 
## 🔴 CRITICAL: Documentation Logging Rule
 
Every session in this project MUST maintain a running tutorial log.
 
**You are required to:**
1. Keep a file called `TUTORIAL.md` in the project root at all times
2. After every meaningful action in this project, append a new entry to `TUTORIAL.md`
3. Write entries as if teaching a complete beginner how to use Claude Code
4. Never delete existing entries — only append
 
---
 
## 📓 TUTORIAL.md — What to Log
 
Every entry in `TUTORIAL.md` must include:
 
- **The prompt** the user gave you (copy it exactly)
- **What you did** in plain English (no jargon)
- **The command or code** you ran (in a code block)
- **What happened** — the result or output
- **What the learner should understand** from this step
 
### Entry format to use:
 
```markdown
---
 
## Step N — [Short Title]
 
### 🗣️ Prompt
> "[exact prompt the user typed]"
 
### 🔍 What Claude Did
[Plain English explanation of the action taken]
 
### 💻 Command / Code
```bash
[command or code block]
```
 
### ✅ Result
[What happened — output, error, or outcome]
 
### 📚 What to Learn
[Key concept this step teaches — written for a beginner]
```
 
---
 
## 📄 TUTORIAL.md — Structure
 
The file must always have this structure:
 
```markdown
# Claude Code Tutorial — RAG Chatbot Project
## Learn Claude Code Step by Step Using a Real Project
 
> This tutorial was auto-generated during a live Claude Code session.
> Every prompt, action, and result was logged in real time.
 
## Project Overview
[Brief description of the RAG chatbot project]
 
## Prerequisites
[What the reader needs before starting]
 
## Steps
[All logged steps go here in order]
 
## Key Concepts Covered
[Running list of concepts the tutorial has covered so far]
```
 
---
 
## 🧠 Documentation Style Rules
 
Write TUTORIAL.md entries as if publishing on:
- **Medium** (clear, narrative, beginner-friendly)
- **GitHub README** (structured, scannable, with code blocks)
 
Rules:
- Use plain English — no assumption of prior Claude Code knowledge
- Explain WHY before HOW
- Every code block must have a comment explaining what it does
- If something went wrong and was fixed, document the error AND the fix — errors are valuable teaching moments
- Use emojis for section headers to make it scannable
- Keep each step self-contained — a reader should be able to jump to any step
 
---
 
## 🏗️ Project Context
 
**Project:** Course Materials RAG System
**Stack:** Python, FastAPI, ChromaDB, Anthropic Claude API, UV package manager
**What it does:** A RAG (Retrieval-Augmented Generation) chatbot that answers questions about Anthropic course materials using vector search + Claude
 
**Key files:**
- `backend/app.py` — FastAPI server
- `backend/config.py` — Configuration (model, chunk size, etc.)
- `backend/.env` — API keys (never commit this)
- `backend/chroma_db/` — Vector database storage
- `run.sh` — Start the server
 
---
 
## ⚠️ Environment Rules
 
- Never commit `.env` files
- API key goes in `backend/.env` as `ANTHROPIC_API_KEY=sk-ant-...`
- Always restart the server after changing `.env`
- Kill old server processes before restarting: `taskkill /F /IM python.exe`
- Model in use: `claude-sonnet-4-20250514` (set in `backend/config.py`)
 
---
 
## 🚫 Behaviour Rules
 
- Do NOT open new VS Code windows — use the integrated terminal only
- Do NOT use the `code` CLI to open files
- Do NOT run long-running background commands without warning
- Always show test results in the terminal, not in a new window
- For server operations: remind the user to start the server manually, then run curl tests
 
---
 
## 📝 Session Start Checklist
 
At the start of every session:
1. Read `TUTORIAL.md` to understand where we left off
2. Note the last step number so you continue from the right place
3. Confirm the server is running before running any API tests
4. Append a "Session Start" entry to `TUTORIAL.md` with today's date