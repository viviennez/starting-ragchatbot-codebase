# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Installing Dependencies
```bash
# Install UV package manager (if needed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync
```

### Environment Setup
Create a `.env` file in the root directory:
```bash
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

### Git commit instruction
Never put Claude as an author of the commit.

## Architecture Overview

### Core Design: Tool-Augmented RAG System

This is **NOT a traditional RAG system**. Instead of pre-retrieving context before every LLM call, this uses **Anthropic's tool calling** to let Claude decide when to search course materials. The AI controls retrieval, not the application.

**Query Flow:**
```
User Query → FastAPI → RAGSystem → AIGenerator (Claude with tools available)
  → Claude decides: Need search?
  → If yes: Calls search_course_content tool
  → ToolManager executes → VectorStore searches ChromaDB
  → Tool results → Claude generates final answer
```

### Component Architecture

**RAGSystem** (`rag_system.py`) is the central orchestrator that coordinates:
- **DocumentProcessor**: Parses course files, chunks text (800 chars, 100 overlap, sentence-aware)
- **VectorStore**: Manages ChromaDB with dual collections (see below)
- **AIGenerator**: Handles Claude API with tool calling loop
- **SessionManager**: In-memory conversation history (max 2 exchanges)
- **ToolManager**: Registers and executes tools (currently: `search_course_content`)

All components receive dependencies via the `Config` object.

### Dual-Collection Vector Store (Critical)

ChromaDB uses **two separate collections**:

1. **course_catalog**: Course metadata only
   - IDs: Course titles (for deduplication)
   - Content: Title, instructor, links, lessons JSON
   - Purpose: Fuzzy course name resolution before content search

2. **course_content**: Actual course text chunks
   - IDs: `{course_title}_{chunk_index}`
   - Content: Context-enhanced chunks
   - Metadata: `course_title`, `lesson_number`, `chunk_index`
   - Purpose: Semantic content retrieval

**Why two collections?** Enables fuzzy matching of course names (e.g., "python intro" → "Introduction to Python") before searching content.

### Context Enhancement Strategy

Every chunk is prefixed with course/lesson context before embedding:
```
"Course {course_title} Lesson {lesson_number} content: {original_text}"
```

This embeds course/lesson identity directly into the vector representation, improving cross-course semantic search.

### Document Processing Pipeline

1. **Parse metadata** from first 3-4 lines (regex: "Course Title:", "Course Link:", "Course Instructor:")
2. **Split into lessons** by regex: `^Lesson\s+(\d+):\s*(.+)$`
3. **Chunk each lesson** using sentence-aware splitting:
   - Splits on sentence boundaries (`.!?` followed by space + capital letter)
   - Respects abbreviations (doesn't split on "Dr." or "U.S.")
   - 800 char chunks with 100 char overlap
   - Never breaks mid-sentence
4. **Add context prefix** to each chunk
5. **Store in ChromaDB** with metadata

Expected document format:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[content...]

Lesson 1: [title]
...
```

### AI System Prompt Behavior

Located in `ai_generator.py:8-30`. Key constraints:
- **Maximum one search per query** (prevents search loops)
- General knowledge questions → no search needed
- Course-specific questions → search first, then answer
- No meta-commentary (avoid "based on search results")
- Temperature: 0 (deterministic responses)
- Max tokens: 800

### Session Management

- **In-memory only** (dict: `{session_id: List[Message]}`)
- Rolling window: Keeps last 4 messages (2 conversation pairs)
- Auto-created on first query if not provided
- Not scalable beyond single instance

### Search Resolution Logic

Two-phase process in `VectorStore.search()`:

1. **If `course_name` provided**: Query `course_catalog` to resolve fuzzy name → exact title
2. **Build metadata filter**:
   - Both course + lesson: `{"$and": [{"course_title": ...}, {"lesson_number": ...}]}`
   - Only course: `{"course_title": ...}`
   - Only lesson: `{"lesson_number": ...}`
   - Neither: No filter (global search)
3. **Query `course_content`** with filter + semantic similarity
4. **Return top N results** (default: 5)

## Important Implementation Details

### Deduplication on Startup
- `RAGSystem.add_course_folder()` checks existing course titles before processing
- Uses `clear_existing=False` by default (incremental, not rebuild)
- Course title is the unique identifier

### Sentence-Aware Chunking
The chunking algorithm in `DocumentProcessor.chunk_text()`:
- Builds chunks sentence-by-sentence (respects semantic units)
- Calculates overlap at sentence level (not character level)
- Ensures forward progress even if single sentence exceeds chunk_size
- Regex handles common abbreviations: `(?<!\w\.\w.)(?<![A-Z][a-z]\.)(?<=\.|\!|\?)\s+(?=[A-Z])`

### Tool Execution Flow
- Single-turn tool execution (no multi-step agentic loops)
- Tool results injected as user message for Claude's final response
- Sources tracked via `last_sources` attribute in `CourseSearchTool`
- Tool manager supports multiple tools, but currently only one is registered

### Configuration Defaults
From `config.py`:
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results per query
- `MAX_HISTORY`: 2 conversation exchanges
- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514
- `EMBEDDING_MODEL`: all-MiniLM-L6-v2 (384-dimensional)

### FastAPI Startup Behavior
- `on_event("startup")` loads documents from `../docs` folder
- Silently handles errors during initial load
- Uses incremental loading (checks for duplicates)

## Working with This Codebase

### Adding New Course Documents
1. Place `.txt` files in `/docs` folder with expected format
2. Restart application - documents auto-load on startup
3. Check `/api/courses` endpoint to verify ingestion

### Modifying the System Prompt
Edit `ai_generator.py:8-30`. Critical: This prompt enforces one-search-per-query constraint.

### Changing Chunking Strategy
Modify `DocumentProcessor.__init__()` parameters or `chunk_text()` logic in `document_processor.py`. Remember: Overlap is calculated at sentence level, not character level.

### Adding New Tools
1. Define tool class in `search_tools.py` (inherit from base pattern)
2. Register in `ToolManager.__init__()`
3. Update system prompt in `ai_generator.py` with tool usage instructions

### Debugging Search Results
Enable logging or add print statements in:
- `VectorStore.search()`: See exact ChromaDB queries and results
- `CourseSearchTool.execute()`: See tool parameters and retrieved chunks
- `AIGenerator._execute_tool()`: See tool execution flow

### Testing Changes
**Current state**: No test infrastructure exists. When adding features, manually test via:
1. Web UI at http://localhost:8000
2. API docs at http://localhost:8000/docs (interactive Swagger UI)
3. Direct API calls: `curl -X POST http://localhost:8000/api/query -H "Content-Type: application/json" -d '{"query": "test question"}'`

## API Reference

### POST /api/query
**Request:**
```json
{
  "query": "What are variables in Python?",
  "session_id": "session_1"  // optional
}
```

**Response:**
```json
{
  "answer": "Variables are...",
  "sources": ["Introduction to Python - Lesson 1"],
  "session_id": "session_1"
}
```

### GET /api/courses
**Response:**
```json
{
  "total_courses": 4,
  "course_titles": ["Introduction to Python", ...]
}
```

## Known Limitations

- **No persistence for sessions**: In-memory only, lost on restart
- **No authentication**: Open to anyone with access
- **No rate limiting**: Vulnerable to abuse
- **No logging framework**: Debugging production issues is difficult
- **No prompt caching**: Not using Anthropic's prompt caching feature
- **Single instance only**: Session storage prevents horizontal scaling
- **File-based vector store**: ChromaDB persistence is local, not distributed

## Key Files Reference

**Core orchestration:**
- `backend/rag_system.py`: Main entry point, coordinates all components
- `backend/app.py`: FastAPI endpoints, startup logic

**RAG pipeline:**
- `backend/ai_generator.py`: Claude API client, tool calling loop, system prompt
- `backend/search_tools.py`: Tool definitions and execution
- `backend/vector_store.py`: ChromaDB interface, dual-collection logic

**Document processing:**
- `backend/document_processor.py`: File parsing, chunking algorithm
- `backend/models.py`: Data models (Course, Lesson, CourseChunk)

**Supporting:**
- `backend/session_manager.py`: Conversation history
- `backend/config.py`: Configuration management

**Frontend:**
- `frontend/index.html`, `frontend/script.js`, `frontend/style.css`
