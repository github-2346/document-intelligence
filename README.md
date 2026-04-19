# Document Intelligence Platform

> AI-powered book knowledge platform with semantic search, RAG-based Q&A, automated scraping.

## Tech Stack
```
| Layer | Technology |

| **Backend** | Django 4.2 · Django REST Framework 3.15 |
| **Database** | MySQL 8.0 (structured data) |
| **Vector Store** | FAISS (default) · ChromaDB (optional) |
| **AI — LLM** | OpenAI GPT-4o-mini · LM Studio (local) |
| **AI — Embeddings** | OpenAI text-embedding-3-small · sentence-transformers fallback |
| **Async Queue** | Celery 5.4 · Redis 7 |
| **Caching** | django-redis |
| **Scraping** | Selenium 4 · webdriver-manager |
| **Frontend** | Next.js 14 · React 18 · Tailwind CSS 3 |
| **HTTP Client** | Axios · SWR |
| **Containerisation** | Docker · Docker Compose |
```
---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        Frontend (Next.js)               │
│   /            /books       /books/:id       /chat      │
└────────────────────────┬────────────────────────────────┘
                         │ HTTP (Axios)
┌────────────────────────▼────────────────────────────────┐
│                  Django REST Framework                   │
│  /api/books/   /api/rag/query/   /api/scraping/trigger/ │
└──────┬──────────────────┬──────────────────┬────────────┘
       │                  │                  │
  ┌────▼────┐       ┌─────▼──────┐    ┌─────▼──────┐
  │  MySQL  │       │   Celery   │    │   Redis    │
  │  Books, │       │  (Scraping,│    │ (Cache +   │
  │ Chat,   │       │  Embeddings│    │  Broker)   │
  │ Insights│       │  Insights) │    └────────────┘
  └─────────┘       └─────┬──────┘
                          │
              ┌───────────▼────────────┐
              │     AI Services        │
              │  OpenAI / LM Studio    │
              │  Embeddings + LLM      │
              └───────────┬────────────┘
                          │
              ┌───────────▼────────────┐
              │     Vector Store       │
              │   FAISS / ChromaDB     │
              └────────────────────────┘
```


## Project Structure

```
document-intelligence/
├── .env.example
├── docker-compose.yml
│
├── backend/
│   ├── Dockerfile
│   ├── manage.py
│   ├── requirements.txt
│   ├── config/
│   │   ├── __init__.py         
│   │   ├── celery.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── apps/
│   │   ├── books/
│   │   │   ├── models.py       
│   │   │   ├── serializers.py
│   │   │   ├── views.py        
│   │   │   ├── tasks.py        
│   │   │   ├── admin.py
│   │   │   └── management/commands/seed_sample_data.py
│   │   ├── rag/
│   │   │   ├── services/
│   │   │   │   ├── chunker.py      # Sentence-aware semantic chunker
│   │   │   │   ├── pipeline.py     # Full RAG pipeline
│   │   │   │   └── vector_store.py # FAISS + ChromaDB abstraction
│   │   │   ├── views.py        # Query + Reindex endpoints
│   │   │   └── tasks.py        # Celery: embedding generation
│   │   ├── scraping/
│   │   │   ├── scraper.py      # Selenium BookScraper
│   │   │   ├── views.py        # Trigger + Status
│   │   │   └── tasks.py        # Celery: full scrape pipeline
│   │   └── chat/
│   │       ├── models.py       # ChatSession, ChatMessage
│   │       ├── serializers.py
│   │       └── views.py        # Query + History + Clear
│   └── utils/
│       └── ai_client.py       
│
└── frontend/
    ├── Dockerfile
    ├── next.config.js
    ├── tailwind.config.js
    ├── pages/
    │   ├── _app.jsx
    │   ├── _document.jsx      
    │   ├── index.jsx           
    │   ├── about.jsx
    │   ├── chat.jsx            
    │   └── books/
    │       ├── index.jsx       
    │       └── [id].jsx        
    ├── components/
    │   ├── layout/
    │   │   ├── Navbar.jsx
    │   │   ├── Hero.jsx
    │   │   └── Footer.jsx
    │   └── ui/
    │       ├── BookCard.jsx
    │       ├── ChatInterface.jsx
    │       ├── InsightPanel.jsx
    │       └── Skeleton.jsx
    ├── hooks/
    │   └── useData.js          
    └── services/
        └── api.js             
```

---

## Setup Instructions

### Prerequisites

- Python 3.11+
- Node.js 20+
- MySQL 8.0
- Redis 7
- Google Chrome (for Selenium scraping)
- An OpenAI API key **or** LM Studio running locally

---

### MySQL Setup

```bash
# Log in to MySQL
mysql -u root -p

# Create database and user
CREATE DATABASE document_intelligence CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'diuser'@'localhost' IDENTIFIED BY 'dipassword';
GRANT ALL PRIVILEGES ON document_intelligence.* TO 'diuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### Redis Setup

```bash
# macOS
brew install redis && brew services start redis

# Ubuntu/Debian
sudo apt install redis-server && sudo systemctl start redis

# Verify
redis-cli ping   # → PONG
```

---

### Backend Setup

```bash
cd backend

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp ../.env.example ../.env
# Edit .env with your MySQL credentials and OpenAI API key

# Export env vars (or use python-dotenv)
export $(cat ../.env | grep -v ^# | xargs)

# Run migrations
python manage.py migrate

# Create Django superuser (optional)
python manage.py createsuperuser

# Seed sample data (no scraping needed to test immediately)
python manage.py seed_sample_data

# Start development server
python manage.py runserver 0.0.0.0:8000
```

---

### Celery Setup

Open two additional terminal windows (with venv activated and env vars exported):

```bash
# Terminal 2 — Celery worker
cd backend
celery -A config worker --loglevel=info --concurrency=2

# Terminal 3 — Celery beat scheduler (optional, for periodic tasks)
cd backend
celery -A config beat --loglevel=info
```

---

### Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Configure environment
echo "NEXT_PUBLIC_API_URL=http://localhost:8000" > .env.local

# Start development server
npm run dev
# → http://localhost:3000
```

---

### Docker (Full Stack)

The easiest way to run everything:

```bash
# From project root
cp .env.example .env
# Edit .env with your OPENAI_API_KEY and a strong DJANGO_SECRET_KEY

docker compose up --build

# First-time only: run migrations and seed data
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py seed_sample_data
```


---

## Environment Variables

Copy `.env.example` to `.env` and fill in the values.

---

## API Documentation

### Books

#### `GET /api/books/`
List all books (paginated).

**Query params:**
| Param | Type | Description |
|---|---|---|
| `search` | string | Filter by title or author |
| `sort` | string | `rating`, `-rating`, `title`, `-title`, `created_at`, `-created_at` |
| `genre` | string | Filter by AI-generated genre |
| `page` | int | Page number (default: 1) |
| `page_size` | int | Results per page (default: 20, max: 100) |


---

#### `GET /api/books/<id>/`
Retrieve a single book with full detail, AI insights, and top-5 recommendations.

---

#### `GET /api/books/<id>/recommendations/`
Get up to 10 recommended books for a given book.

---

#### `POST /api/books/upload/`
Manually add a book and trigger background AI processing.


**Response:** `201 Created` with full book object.

---

### RAG

#### `POST /api/rag/query/`

### Chat

#### `POST /api/chat/query/`
RAG-powered Q&A with persistent session history.


#### `GET /api/chat/history/<session_key>/`
Retrieve all messages in a session.

#### `DELETE /api/chat/history/<session_key>/clear/`
Clear all messages in a session.

---

### Scraping

#### `POST /api/scraping/trigger/`
Start a background scrape job.

#### `GET /api/scraping/status/<task_id>/`
Poll scrape job status.

---

## RAG Pipeline

The full pipeline executes in this order:

```
1. CHUNK
   Book description → sentence-aware splitter
   → overlapping chunks of ~500 characters

2. EMBED
   Each chunk → OpenAI text-embedding-3-small
   → 1536-dimensional float vector

3. STORE
   Vectors → FAISS IndexFlatIP (cosine similarity)
   Metadata (book_id, title, chunk_index) → JSON sidecar
   Chunk text → MySQL BookEmbedding table

4. QUERY (at runtime)
   User question → embed → 1536-d vector
   → FAISS similarity search → top-K chunks

5. RETRIEVE
   Fetch chunk text from MySQL by vector_id
   Build context string with source attribution

6. GENERATE
   Context + question → GPT-4o-mini prompt
   → Grounded answer with citations

7. CACHE
   MD5(question + book_id) → Redis (1 hour TTL)
   Subsequent identical queries served from cache
```

## Testing

### Backend

```bash
cd backend
source venv/bin/activate

# Run all tests
python manage.py test

# Test individual apps
python manage.py test apps.books
python manage.py test apps.rag
python manage.py test apps.chat

# Verify the RAG pipeline manually
python manage.py shell
>>> from apps.rag.services.pipeline import RAGPipeline
>>> p = RAGPipeline()
>>> p.query("What is 1984 about?")

# Verify AI client
>>> from utils.ai_client import get_ai_client
>>> client = get_ai_client()
>>> client.complete("Say hello in one word.")
```

### Frontend

```bash
cd frontend
npm run lint        
npm run build       
```

### Celery

```bash
# Verify a task runs correctly
cd backend
python manage.py shell
>>> from apps.books.tasks import generate_insights_for_book
>>> result = generate_insights_for_book.delay(1)
>>> result.get(timeout=30)
```

## 📸 UI Screenshots

### 🏠 Home Page
<p align="center">
  <img src="docs/screenshots/home.png" width="800"/>
</p>

### 📚 Books Page
<p align="center">
  <img src="docs/screenshots/books.png" width="800"/>
</p>

### 🤖 Chat Page
<p align="center">
  <img src="docs/screenshots/chat.png" width="800"/>
</p>

---

*Built with Django, Next.js, OpenAI, FAISS, Celery, Redis, and MySQL.*
