# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

Install dependencies (from project root):
```bash
uv sync
```

Copy `.env.example` to `.env` in the project root. The `.env` file is loaded by `backend/config.py` via `python-dotenv`.

## Running the Application

The server must be started from the `backend/` directory (not the project root), because `app.py` uses relative paths like `../docs` and `../frontend`.

```bash
cd backend && uv run uvicorn app:app --reload --port 8000
```

On Windows, use Git Bash. The app is served at `http://localhost:8000`. Interactive API docs are at `http://localhost:8000/docs`.

## Architecture

This is a full-stack RAG (Retrieval-Augmented Generation) chatbot. The backend is a FastAPI app that also serves the frontend as static files.

### Request Flow

1. User submits a query via the frontend (`frontend/script.js` → `POST /api/query`)
2. `app.py` passes the query to `RAGSystem.query()`
3. `RAGSystem` sends the query to `AIGenerator` along with available tools and conversation history
4. The LLM (via Ollama's OpenAI-compatible API) decides whether to call the `search_course_content` tool
5. If tool is called, `ToolManager` dispatches to `CourseSearchTool`, which queries `VectorStore`
6. `VectorStore` does a two-step search: semantic course name resolution (via `course_catalog` collection), then content search (via `course_content` collection)
7. Search results are returned as a tool result; the LLM generates the final answer
8. Sources are extracted from `CourseSearchTool.last_sources` and returned alongside the answer

### Key Components

- **`backend/rag_system.py`** — Central orchestrator; wires together all components
- **`backend/ai_generator.py`** — LLM client using OpenAI-compatible API (Ollama); handles the tool-use loop (initial call → tool execution → follow-up call)
- **`backend/vector_store.py`** — ChromaDB wrapper with two collections: `course_catalog` (course titles/metadata) and `course_content` (chunked lesson text)
- **`backend/search_tools.py`** — `CourseSearchTool` (OpenAI function-calling tool definition + execution) and `ToolManager` (registry/dispatcher)
- **`backend/document_processor.py`** — Parses `.txt` course files into `Course`/`Lesson`/`CourseChunk` models and splits content into overlapping chunks
- **`backend/models.py`** — Pydantic/dataclass models: `Course`, `Lesson`, `CourseChunk`
- **`backend/session_manager.py`** — In-memory conversation history keyed by session ID
- **`backend/config.py`** — All tunable parameters (model, chunk size, embedding model, ChromaDB path); reads `OLLAMA_BASE_URL` and `OLLAMA_MODEL` from env with defaults

### Course Document Format

Files in `docs/` must follow this structure for the parser to extract metadata:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 1: [lesson title]
Lesson Link: [url]
[lesson content...]

Lesson 2: [lesson title]
...
```

### Data Persistence

ChromaDB is persisted to `backend/chroma_db/`. On startup, `app.py` loads all `.txt`/`.pdf`/`.docx` files from `docs/` but skips courses already in the vector store (deduplication by title). To force a full re-index, delete `backend/chroma_db/`.

### Frontend

Static single-page app in `frontend/` served directly by FastAPI. No build step. The `style.css` and `script.js` versioning (`?v=9`) is manual cache-busting — increment when making frontend changes.
