# Architecture Notes — AI Tutor Case Study

Public-safe architecture summary. Source code is in a private repository.

## System Shape

AI Tutor is a full-stack RAG platform with three main layers:

```text
┌─────────────────────────────────────┐
│        Next.js Frontend             │
│  Dashboard / Chat / File Upload     │
└──────────────┬──────────────────────┘
               │ REST + JWT Auth
┌──────────────▼──────────────────────┐
│        FastAPI Backend              │
│  Auth / Subjects / Docs / Chat      │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         RAG Pipeline                │
│  Parse → Chunk → Embed → Search     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Infrastructure              │
│  Supabase / pgvector / OpenAI       │
│  Celery / Redis / Render / Vercel   │
└─────────────────────────────────────┘
```

## RAG Pipeline Detail

### Ingestion

1. User uploads document (PDF, DOCX, PPTX, MP3)
2. File validated (type, size, ownership)
3. Uploaded to Supabase Storage with user-scoped path
4. Database record created

### Processing (background task)

1. **PDF Processing (PyMuPDF)**
   - Extract text page by page
   - Extract embedded images with size filtering
   - Skip decorative/small images (<50KB)
   - Limit to top 3 largest images per page

2. **Vision Analysis (GPT-4 Vision)**
   - Analyze diagrams, charts, illustrations
   - Produce structured descriptions
   - MD5-based deduplication cache
   - Low detail mode + 150 token cap for cost control

3. **Text Chunking**
   - RecursiveCharacterTextSplitter
   - 512 character chunks, 50 character overlap
   - Preserves page/source metadata

4. **Embedding Generation**
   - text-embedding-3-small model
   - Chunks stored with vector embeddings in Supabase

### Retrieval + Generation

1. User question → embedded to vector
2. Cosine similarity search via pgvector `match_chunks` RPC
3. Top-K chunks (default 5, similarity threshold 0.5)
4. Context assembled with source annotations
5. GPT-4o generates answer with retrieved context
6. Response includes sources with similarity scores

## API Surface

```
POST   /api/auth/signup        — Create account
POST   /api/auth/signin        — Login
GET    /api/auth/me            — Current user

POST   /api/subjects/          — Create subject
GET    /api/subjects/          — List user's subjects
GET    /api/subjects/:id       — Get subject
PUT    /api/subjects/:id       — Update subject
DELETE /api/subjects/:id       — Delete subject

POST   /api/documents/upload   — Upload document
GET    /api/documents/subject/:id — List documents by subject
GET    /api/documents/         — List all user documents
DELETE /api/documents/:id      — Delete document

POST   /api/chat/              — Send message (RAG)

GET    /api/health             — Health check
```

## Frontend Component Tree

```
App
├── Auth Page
│   ├── LoginForm
│   └── SignupForm
├── Dashboard Page
│   ├── Navbar (user info, sign out)
│   ├── Sidebar (subjects list, add/delete)
│   ├── ChatInterface (messages, sources, input)
│   └── FilesPanel (upload, file list, delete)
└── Auth Callback (token handling)
```

## Database Schema (Supabase)

```
subjects
  id, user_id, name, description, color, icon, document_count

documents
  id, user_id, subject_id, filename, file_type, file_url,
  file_size, processed, created_at

chunks (pgvector)
  id, document_id, subject_id, content, embedding (vector),
  page_number, metadata

chat_messages
  id, user_id, subject_id, role, content, sources, tokens_used
```

## Security Design

- All API routes protected by JWT bearer token dependency
- User-scoped queries on every endpoint (subjects, documents, chat)
- File ownership verification before upload/delete
- Supabase Row-Level Security for database access
- Environment variables for all secrets (never tracked)
- `.env.example` template with no real values
