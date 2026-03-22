# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

**Windows users must use Git Bash**, not PowerShell or CMD.

```bash
# Install dependencies (first time only)
uv sync

# Set up environment
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY

# Start the server
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

App runs at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

On startup, the server automatically ingests all `.txt`/`.pdf`/`.docx` files from `../docs/` into ChromaDB, skipping any courses already present.

## Architecture

This is a RAG chatbot with a FastAPI backend and vanilla JS frontend. The backend modules are imported relative to the `backend/` directory — all Python imports assume that as the working directory.

### Request flow

```
POST /api/query
  → RAGSystem.query()
    → SessionManager (fetch history)
    → AIGenerator.generate_response()   ← 1st Claude call (tool_choice: auto)
      [if Claude calls search_course_content tool]
      → ToolManager.execute_tool()
        → CourseSearchTool.execute()
          → VectorStore.search()
            → ChromaDB cosine similarity (top 5 chunks)
      → AIGenerator._handle_tool_execution()  ← 2nd Claude call (no tools)
    → SessionManager (save exchange)
  → return (answer, sources)
```

### Key design decisions

**Two Claude API calls per tool-use query**: The first call decides whether to search; if so, results are fed back in a second call for synthesis. Queries Claude can answer from general knowledge skip the tool entirely.

**`course_title` is the unique ID** for ChromaDB collections. Duplicate courses are detected by title at ingestion time and skipped.

**Conversation history is injected into the system prompt** (not as message history). `SessionManager` stores up to `MAX_HISTORY=2` exchanges (4 messages), formatted as plain text appended to the system prompt string.

**Two ChromaDB collections**:
- `course_catalog` — one document per course (title, instructor, lesson list as JSON string). Used only for fuzzy course name resolution.
- `course_content` — one document per chunk. Used for semantic search. IDs are `{course_title}_{chunk_index}`.

**Tool registration is extensible**: `ToolManager` accepts any class implementing the `Tool` ABC (`get_tool_definition()` + `execute()`). Adding a new tool only requires registering it in `RAGSystem.__init__`.

### Document format expected by `DocumentProcessor`

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <title>
Lesson Link: <url>
<content lines...>

Lesson 1: <title>
<content lines...>
```

Text is split into sentence-based chunks (~800 chars) with ~100 char overlap. The first chunk of each lesson is prefixed with `"Lesson N content: "` to preserve retrieval context.

### Configuration (`backend/config.py`)

All tuneable parameters live in the `Config` dataclass. No CLI flags — change values directly in `config.py`:

| Setting | Default | Effect |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model used |
| `CHUNK_SIZE` | `800` | Max chars per chunk |
| `CHUNK_OVERLAP` | `100` | Overlap between chunks |
| `MAX_RESULTS` | `5` | Chunks returned per search |
| `MAX_HISTORY` | `2` | Conversation exchanges remembered |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB persistence directory (relative to `backend/`) |

To rebuild the vector store from scratch, pass `clear_existing=True` to `rag_system.add_course_folder()`, or delete the `backend/chroma_db/` directory and restart.
