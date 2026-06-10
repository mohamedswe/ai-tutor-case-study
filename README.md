# AI Tutor — RAG-Powered Learning Platform

## What this is

AI Tutor is a full-stack Retrieval-Augmented Generation (RAG) platform that lets students upload study materials and ask questions against their own content. Instead of a generic chatbot, users get a tutor that understands their actual course documents — PDFs, presentations, and more.

The platform ingests documents, processes them through a multi-modal pipeline (text + images/diagrams), generates embeddings, and retrieves the most relevant chunks to ground AI responses in real course material.

**I built this project solo** — full-stack from architecture through implementation, covering the backend RAG pipeline, document processing, vector search, API layer, authentication, and the Next.js dashboard interface.

## Architecture

```text
Next.js Dashboard
  |-- subject management (create / organize by course)
  |-- drag-and-drop file upload (PDF, DOCX, PPTX, MP3)
  |-- chat interface with source citations
  v
FastAPI Backend
  |-- auth API (signup / signin / JWT)
  |-- documents API (upload / list / delete)
  |-- chat API (RAG query with context retrieval)
  |-- subjects API (CRUD)
  v
RAG Pipeline
  |-- PDF processor (PyMuPDF text + image extraction)
  |-- GPT-4 Vision (diagram/image analysis & description)
  |-- text chunking (RecursiveCharacterTextSplitter)
  |-- embedding generation (OpenAI text-embedding-3-small)
  |-- vector similarity search (pgvector via Supabase RPC)
  |-- context construction + LLM response generation
  v
Infrastructure
  |-- Supabase (PostgreSQL + pgvector + auth + storage)
  |-- OpenAI (embeddings + chat + vision)
  |-- Celery / Redis (background job queue — ready)
  |-- Render (backend) / Vercel (frontend)
```

## Tech Stack

**Backend:** Python, FastAPI, Pydantic, LangChain, OpenAI API, PyMuPDF, Supabase, Celery, Redis
**Frontend:** TypeScript, Next.js 16, React 19, TanStack Query, Zustand, Tailwind CSS
**AI/ML:** text-embedding-3-small, GPT-4o (chat), GPT-4 Vision (diagram analysis), pgvector
**Deployment:** Docker, Render, Vercel

## Engineering Highlights

### 1. Multi-Modal Document Processing

The document ingestion pipeline handles more than just text. It extracts both textual content AND embedded images/diagrams from PDFs, then analyzes visual elements using GPT-4 Vision to produce structured descriptions that become searchable alongside the text.

Key implementation details:
- **PyMuPDF** extracts text page-by-page with page number tracking
- **GPT-4 Vision** analyzes diagrams, charts, and educational illustrations
- **Image deduplication** via MD5 caching prevents redundant API calls on re-used images
- **Smart filtering** skips too-small images (icons/decorations) and too-large images (full-page renders)
- Configurable image limits per page prevent token waste

> Engineering decision: using "low" detail mode on GPT-4 Vision with 150 max tokens for image descriptions keeps costs controlled while still producing searchable diagram summaries.

### 2. RAG Pipeline with pgvector

The retrieval system combines LangChain's text splitting with Supabase's pgvector extension for production-grade vector search.

Pipeline:
1. Document text is split with `RecursiveCharacterTextSplitter` (512 char chunks, 50 char overlap)
2. Each chunk is embedded via `text-embedding-3-small`
3. Embeddings stored in Supabase with pgvector indexing
4. User questions are embedded and matched via cosine similarity
5. Top-K relevant chunks are retrieved and assembled into context
6. Context + conversation history fed to GPT-4o for grounded answers

The vector search uses a custom PostgreSQL function (`match_chunks`) called via Supabase RPC — proper SQL-level vector operations rather than in-memory approximate nearest neighbor hacks.

### 3. Full Authentication System

Complete auth flow with JWT token management:
- Email/password signup and signin endpoints
- Supabase Auth integration for secure credential handling
- JWT-based route protection via FastAPI dependency injection
- Token refresh and session persistence
- Zustand state management on the frontend with auth callbacks

### 4. Production-Ready Backend Design

The backend is structured for real deployment:

- **Clean service layer separation** — API routes delegate to service classes, keeping route handlers thin
- **Background task support** — Celery + Redis integrated for async document processing
- **File validation** — type checking (MIME + extension), size limits (100MB), user ownership verification
- **Subject-based multi-tenancy** — users organize documents by subject; all queries scoped to user + subject
- **Supabase Storage** for document files with user-scoped paths
- **Health check endpoint** for deployment monitoring
- **CORS configuration** via environment variables
- **Docker support** with Dockerfile for containerized deployment
- **Render.yaml** with proper env var handling (sync:false for secrets, generateValue for app secret)

### 5. Dashboard UI

The frontend is a functional Next.js dashboard with:
- **Subject sidebar** — create, select, and delete subjects with color coding
- **Chat interface** — message history, loading states, source citations with similarity scores, token usage display
- **File panel** — drag-and-drop upload with progress indicators, file type icons, delete with confirmation
- **Responsive layout** — sidebar + chat center + files panel right, with proper empty/null states
- **Auth flow** — login/signup forms, protected routes, token persistence

## Why This Project Matters

A generic chatbot doesn't know your course material. AI Tutor demonstrates the full RAG workflow:

```
Student uploads lecture slides
  → Backend extracts text AND analyzes diagrams with vision
  → Content is chunked, embedded, and indexed
  → Student asks "explain the Krebs cycle diagram from slide 14"
  → System retrieves relevant chunks including the vision-analyzed diagram description
  → GPT-4o generates a grounded answer citing specific document sections
```

This project shows competence across the full stack: document parsing, embedding pipelines, vector databases, API design, authentication, and modern frontend development with React/Next.js.



## Privacy Note

The source code is in a private repository. This public case study describes the architecture, technical decisions, and engineering approach without exposing source code, environment secrets, or deployment details.

## Status

This project is a functional MVP. The next phase would involve deploying live, adding real-time streaming chat responses, and expanding document format support.
