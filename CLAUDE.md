# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
- **Quick start**: `chmod +x run.sh && ./run.sh`
- **Manual start**: `cd backend && uv run uvicorn app:app --reload --port 8000`
- **Prerequisites**: Python 3.13+, uv package manager, Anthropic API key in `.env`

### Package Management
- **Install dependencies**: `uv sync`
- **All dependencies managed via**: `pyproject.toml` with uv as the package manager

### Environment Setup
- Copy `.env.example` to `.env` and add your `ANTHROPIC_API_KEY`
- Application runs on `http://localhost:8000` with API docs at `/docs`

## Architecture Overview

This is a full-stack RAG (Retrieval-Augmented Generation) system for querying course materials using semantic search and AI responses.

### Request Flow Diagram

```text
User Query Flow:
┌─────────────┐    HTTP POST     ┌─────────────┐
│   Frontend  │ ──────────────► │   FastAPI   │
│ (index.html)│                 │   (app.py)  │
└─────────────┘                 └─────────────┘
                                        │
                                        ▼
                            ┌─────────────────────┐
                            │    RAG System       │
                            │  (rag_system.py)    │
                            └─────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
            ┌─────────────┐    ┌─────────────────┐   ┌──────────────┐
            │Session Mgr  │    │   AI Generator  │   │ Vector Store │
            │(session_    │    │ (ai_generator.  │   │ (vector_     │
            │ manager.py) │    │      py)        │   │  store.py)   │
            └─────────────┘    └─────────────────┘   └──────────────┘
                                        │                   │
                                        │ ◄──────────────── │ Semantic
                                        ▼        Tool Calls │ Search
                            ┌─────────────────────┐         │
                            │   Search Tools      │         │
                            │ (search_tools.py)   │ ────────┘
                            └─────────────────────┘
                                        │
                                        ▼
                              ┌─────────────────┐
                              │    ChromaDB     │
                              │  (chroma_db/)   │
                              └─────────────────┘
                                        
                      🌐 EXTERNAL API CALL:
                      AI Generator ──────────────► Anthropic Claude API
                      (with search results + tools)

Response Path: ChromaDB → Vector Store → Search Tools → AI Generator → RAG System → FastAPI → Frontend
```

### Core Architecture Components

**Backend (`/backend/`):**
- `app.py` - FastAPI application with CORS, static file serving, and API endpoints
- `rag_system.py` - Main orchestrator coordinating all components
- `config.py` - Configuration management with environment variables
- `vector_store.py` - ChromaDB integration for vector storage and semantic search
- `ai_generator.py` - Anthropic Claude API integration with tool-calling capabilities
- `document_processor.py` - Document parsing and chunking (PDF, DOCX, TXT)
- `search_tools.py` - Tool-based search system for AI function calling
- `session_manager.py` - Conversation history management
- `models.py` - Data models for Course, Lesson, CourseChunk

**Frontend (`/frontend/`):**
- Static HTML/CSS/JS served by FastAPI
- Simple web interface for querying the RAG system

**Data:**
- `docs/` - Course materials (TXT files automatically loaded on startup)
- `backend/chroma_db/` - ChromaDB persistent storage for vectors and metadata

### Key Design Patterns

**Tool-Based AI Architecture:**
- AI uses function calling to search course content
- `ToolManager` orchestrates available search tools
- `CourseSearchTool` performs vector similarity search
- Maximum one search per query to prevent over-searching

**Modular RAG Pipeline:**
1. Document processing → chunking → vector embedding
2. Query → tool-based search → context retrieval → AI generation
3. Session management for conversation continuity

**Configuration Management:**
- Centralized config in `config.py` with environment variable overrides
- Default models: Claude Sonnet 4, all-MiniLM-L6-v2 embeddings
- Configurable chunk sizes (800 chars), overlap (100 chars), max results (5)

### Database Schema

ChromaDB collections:
- Course metadata collection (course titles, descriptions)
- Content chunks collection (text segments with course/lesson metadata)
- Automatic deduplication based on course titles

### API Endpoints

- `POST /api/query` - Process queries with optional session management
- `GET /api/courses` - Course analytics and statistics
- `/` - Static file serving for frontend
- `/docs` - FastAPI OpenAPI documentation
- always use uv to run the server, do not use pip directly