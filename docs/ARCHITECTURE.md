# Konetou - Architecture & Design Document

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Overview](#2-system-overview)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Component Architecture](#4-component-architecture)
5. [Data Architecture](#5-data-architecture)
6. [RAG Pipeline Design](#6-rag-pipeline-design)
7. [LLM Provider Abstraction](#7-llm-provider-abstraction)
8. [Caching Architecture](#8-caching-architecture)
9. [API Design](#9-api-design)
10. [Frontend Architecture](#10-frontend-architecture)
11. [Deployment Architecture](#11-deployment-architecture)
12. [Security Architecture](#12-security-architecture)
13. [Performance Optimizations](#13-performance-optimizations)
14. [Scalability Considerations](#14-scalability-considerations)
15. [Appendix](#15-appendix)

---

## 1. Executive Summary

### 1.1 Project Overview

**Konetou** is a production-ready Retrieval-Augmented Generation (RAG) system developed by R-Mad Ltd. The system enables intelligent knowledge management by automatically scraping web content, storing it in a vector database, and providing AI-powered question-answering capabilities with source citations.

### 1.2 Key Capabilities

- **Dual LLM Provider Support**: Seamlessly switch between OpenAI (cloud) and Local (Ollama + SentenceTransformers)
- **Automated Web Scraping**: Schedule periodic URL scanning with configurable intervals
- **Vector Search**: ChromaDB-powered semantic similarity search
- **Multi-Tier Caching**: 3-level caching system for 3000x performance improvement
- **Streaming Responses**: Real-time response streaming via Server-Sent Events
- **Production Deployment**: Complete CI/CD pipeline with GitHub Actions

### 1.3 Technology Stack

| Layer | Technology |
|-------|------------|
| **Backend Framework** | FastAPI 0.104.1 |
| **Web Server** | Uvicorn (ASGI) |
| **Vector Database** | ChromaDB 0.4.18 |
| **Relational Database** | SQLite |
| **Frontend Framework** | React 19.2.0 |
| **Styling** | TailwindCSS 3.4.18 |
| **Reverse Proxy** | Nginx |
| **Process Manager** | Systemd |
| **CI/CD** | GitHub Actions |
| **LLM (Cloud)** | OpenAI GPT-4o-mini |
| **LLM (Local)** | Ollama (llama3.2) |
| **Embeddings (Cloud)** | OpenAI text-embedding-3-small |
| **Embeddings (Local)** | SentenceTransformers (all-MiniLM-L6-v2) |

---

## 2. System Overview

### 2.1 System Context Diagram

```
                                    +------------------+
                                    |    End Users     |
                                    +--------+---------+
                                             |
                                             | HTTPS
                                             v
                                    +--------+---------+
                                    |      Nginx       |
                                    | (Reverse Proxy)  |
                                    +--------+---------+
                                             |
                    +------------------------+------------------------+
                    |                                                 |
                    v                                                 v
           +-------+--------+                              +---------+--------+
           |   React SPA    |                              |   FastAPI        |
           |   (Frontend)   |                              |   (Backend API)  |
           +----------------+                              +--------+---------+
                                                                    |
                    +------------------------+------------------------+
                    |                        |                        |
                    v                        v                        v
           +-------+--------+       +-------+--------+       +-------+--------+
           |    SQLite      |       |   ChromaDB     |       |  LLM Provider  |
           | (URL Metadata) |       | (Vector Store) |       | (OpenAI/Local) |
           +----------------+       +----------------+       +----------------+
```

### 2.2 Project Directory Structure

```
konetou-v2/
├── rag-application/
│   ├── backend/
│   │   ├── main.py                    # FastAPI application entry point
│   │   ├── llm_provider_openai.py     # OpenAI provider implementation
│   │   ├── llm_provider_local.py      # Local provider implementation
│   │   ├── requirements.txt           # All dependencies
│   │   ├── requirements-openai.txt    # OpenAI-specific dependencies
│   │   ├── requirements-local.txt     # Local provider dependencies
│   │   ├── .env.example               # Environment variable template
│   │   ├── chroma_data/               # ChromaDB persistent storage
│   │   └── konetou_data.db            # SQLite database
│   └── frontend/
│       ├── src/
│       │   ├── App.js                 # Main React application
│       │   └── index.js               # Application entry point
│       ├── package.json               # Node.js dependencies
│       ├── tailwind.config.js         # TailwindCSS configuration
│       └── build/                     # Production build output
├── .github/
│   └── workflows/
│       └── deploy.yml                 # GitHub Actions CI/CD workflow
├── nginx.conf                         # Nginx reverse proxy configuration
├── deploy.sh                          # Automated deployment script
├── test_performance.sh                # Performance testing script
├── install_gpu_drivers.sh             # NVIDIA GPU driver installer
├── README.md                          # User documentation
├── DEPLOYMENT.md                      # Production deployment guide
└── ARCHITECTURE.md                    # This document
```

---

## 3. High-Level Architecture

### 3.1 Architecture Pattern

Konetou follows a **microservices-inspired monolithic architecture** with clear separation of concerns:

```
+------------------------------------------------------------------+
|                         Client Layer                              |
|  +------------------------------------------------------------+  |
|  |              React Single Page Application                  |  |
|  |  - Query Interface    - URL Management    - Stats Dashboard |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
                                  |
                                  | REST API (JSON)
                                  v
+------------------------------------------------------------------+
|                       API Gateway Layer                           |
|  +------------------------------------------------------------+  |
|  |                    Nginx Reverse Proxy                      |  |
|  |  - SSL Termination  - Static Serving  - Request Routing    |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
                                  |
                                  | HTTP (localhost)
                                  v
+------------------------------------------------------------------+
|                      Application Layer                            |
|  +------------------------------------------------------------+  |
|  |                    FastAPI Backend                          |  |
|  |  +------------------+  +------------------+                 |  |
|  |  | Request Handlers |  | Background Tasks |                 |  |
|  |  +--------+---------+  +--------+---------+                 |  |
|  |           |                     |                           |  |
|  |  +--------v---------+  +--------v---------+                 |  |
|  |  | Caching Layer    |  | Scheduler        |                 |  |
|  |  | (L1/L2/L3)       |  | (APScheduler)    |                 |  |
|  |  +------------------+  +------------------+                 |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
                                  |
        +-------------------------+-------------------------+
        |                         |                         |
        v                         v                         v
+---------------+        +----------------+        +----------------+
| Provider Layer|        |  Data Layer    |        |  Data Layer    |
|  +---------+  |        |  +----------+  |        |  +----------+  |
|  | OpenAI  |  |        |  | ChromaDB |  |        |  | SQLite   |  |
|  | Provider|  |        |  | (Vectors)|  |        |  | (URLs)   |  |
|  +---------+  |        |  +----------+  |        |  +----------+  |
|  +---------+  |        +----------------+        +----------------+
|  | Local   |  |
|  | Provider|  |
|  +---------+  |
+---------------+
```

### 3.2 Design Principles

1. **Provider Abstraction**: LLM providers are abstracted behind a common interface, enabling seamless switching
2. **Stateless API**: Backend maintains no session state; all state is stored in databases
3. **Asynchronous Processing**: Non-blocking I/O operations using Python asyncio
4. **Cache-First Strategy**: Multi-tier caching for optimal performance
5. **Graceful Degradation**: System continues to function even when LLM is unavailable

---

## 4. Component Architecture

### 4.1 Backend Components

#### 4.1.1 FastAPI Application (`main.py`)

The core application orchestrates all functionality:

```python
# Component Structure
main.py
├── Configuration & Initialization
│   ├── Environment Loading (python-dotenv)
│   ├── Logging Configuration
│   ├── LLM Provider Selection
│   └── Database Initialization
├── Database Layer
│   ├── SQLite Connection Management
│   ├── URL CRUD Operations
│   └── Status Tracking
├── Caching Layer
│   ├── L1 Query Cache
│   ├── L2 Embedding Cache
│   └── L3 LLM Response Cache
├── Web Scraping
│   ├── Content Extraction (Trafilatura)
│   ├── Fallback Parser (BeautifulSoup)
│   └── Text Chunking
├── Vector Operations
│   ├── Embedding Generation
│   ├── ChromaDB Storage
│   └── Similarity Search
├── API Endpoints
│   ├── URL Management
│   ├── Query Processing
│   ├── Cache Management
│   └── Statistics
└── Background Services
    ├── Scheduled Crawling (APScheduler)
    └── Graceful Shutdown
```

#### 4.1.2 OpenAI Provider (`llm_provider_openai.py`)

Cloud-based LLM provider implementation:

```python
class OpenAIProvider:
    """
    Attributes:
        client: AsyncOpenAI          # Async API client
        embedding_model: str         # text-embedding-3-small
        chat_model: str              # gpt-4o-mini
        embedding_cache: dict        # Internal cache (500 items)

    Methods:
        generate_embeddings(texts)    # Batch embedding generation
        get_cached_embedding(text)    # Single embedding with caching
        generate_chat_completion()    # LLM response generation
        stream_chat_completion()      # Streaming LLM responses
        clear_cache()                 # Cache management
        get_provider_info()           # Provider metadata
    """
```

**Key Features:**
- Async OpenAI client for non-blocking operations
- Batch processing (100 items per API call)
- Internal embedding cache with LRU eviction
- Streaming support via Server-Sent Events

#### 4.1.3 Local Provider (`llm_provider_local.py`)

Self-hosted LLM provider implementation:

```python
class LocalProvider:
    """
    Attributes:
        embedding_model: SentenceTransformer  # all-MiniLM-L6-v2
        chat_model_name: str                  # llama3.2
        executor: ThreadPoolExecutor          # 4 workers

    Methods:
        generate_embeddings(texts)    # Local embedding generation
        get_cached_embedding(text)    # LRU-cached embedding
        generate_chat_completion()    # Ollama-based generation
        stream_chat_completion()      # Streaming responses
        shutdown()                    # Resource cleanup
    """
```

**Key Features:**
- GPU-accelerated embeddings via PyTorch
- Thread pool for blocking Ollama operations
- functools.lru_cache for embedding caching
- Model pre-warming on initialization

### 4.2 Frontend Components

#### 4.2.1 React Application (`App.js`)

Single-page application structure:

```javascript
RAGApplication
├── State Management (useState)
│   ├── urls[]           // Tracked URLs
│   ├── query            // Current search query
│   ├── answer           // Query response
│   ├── stats            // System statistics
│   └── activeTab        // Current view
├── Effects (useEffect)
│   ├── Initial Data Fetch
│   └── Auto-Refresh (30s interval)
├── API Integration
│   ├── fetchUrls()      // GET /urls
│   ├── fetchStats()     // GET /stats
│   ├── addUrl()         // POST /urls/add
│   ├── removeUrl()      // DELETE /urls/remove
│   ├── refreshUrl()     // POST /urls/refresh
│   └── handleQuery()    // POST /query
└── UI Components
    ├── Header & Stats Cards
    ├── Tab Navigation
    ├── Query Interface
    │   ├── Search Input
    │   └── Answer Display
    │       ├── RelevanceBadge
    │       └── RelevanceBar
    └── URL Management
        ├── Add URL Form
        └── URL List
```

#### 4.2.2 Custom Components

**RelevanceBadge** - Circular progress indicator with gamified scoring:
- 80%+ → Emerald (Excellent)
- 60-79% → Blue (Good)
- 40-59% → Amber (Fair)
- <40% → Orange (Low)

**RelevanceBar** - Animated horizontal progress bar for visual relevance feedback.

---

## 5. Data Architecture

### 5.1 Database Schema

#### 5.1.1 SQLite Schema (`konetou_data.db`)

```sql
CREATE TABLE urls (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    url TEXT UNIQUE NOT NULL,              -- Source URL
    scan_interval_minutes INTEGER DEFAULT 60,  -- Crawl frequency
    status TEXT DEFAULT 'pending',         -- pending|success|error|empty
    last_crawled TEXT,                     -- ISO 8601 timestamp
    last_error TEXT,                       -- Error message if failed
    chunks_count INTEGER DEFAULT 0,        -- Number of stored chunks
    title TEXT,                            -- Extracted page title
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Performance indexes
CREATE INDEX idx_url ON urls(url);
CREATE INDEX idx_status ON urls(status);
```

**URL Status State Machine:**

```
    +----------+
    | pending  |  Initial state
    +----+-----+
         |
         v
    +----+-----+     +-------+
    | success  |<--->| error |  Crawl result
    +----+-----+     +---+---+
         |               |
         v               v
    +----+-----+         |
    |  empty   |<--------+  No content extracted
    +----------+
```

#### 5.1.2 ChromaDB Vector Store

```
Collection: "web_documents"
├── Storage: ./chroma_data (persistent)
├── Embedding Dimensions:
│   ├── OpenAI: 1536 (text-embedding-3-small)
│   └── Local: 384 (all-MiniLM-L6-v2)
└── Document Structure:
    ├── id: MD5(url + chunk_index)
    ├── embedding: float[]
    ├── document: str (chunk text)
    └── metadata:
        ├── url: str
        ├── title: str
        ├── chunk_index: int
        ├── total_chunks: int
        └── timestamp: ISO 8601
```

### 5.2 Data Flow Diagram

```
                           Document Ingestion Flow
+--------+    +----------+    +--------+    +----------+    +----------+
|  URL   |--->|  Scrape  |--->| Chunk  |--->| Embed    |--->| Store    |
| Input  |    | Content  |    | Text   |    | Chunks   |    | Vectors  |
+--------+    +----------+    +--------+    +----------+    +----------+
    |              |              |              |              |
    v              v              v              v              v
 Validate      Extract       500 words/    Generate      ChromaDB
   URL        w/ Trafilatura   50 overlap   embeddings    + SQLite


                             Query Processing Flow
+--------+    +----------+    +----------+    +----------+    +--------+
| Query  |--->| Generate |--->| Vector   |--->| Build    |--->| LLM    |
| Input  |    | Embedding|    | Search   |    | Context  |    | Answer |
+--------+    +----------+    +----------+    +----------+    +--------+
    |              |              |              |              |
    |         L2 Cache       ChromaDB      Truncate to       L3 Cache
    |              |              |         600 chars           |
    +-------> L1 Cache (Full Query Result) <-------------------+
```

---

## 6. RAG Pipeline Design

### 6.1 Document Ingestion Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DOCUMENT INGESTION PIPELINE                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │  Input   │    │  Scrape  │    │  Parse   │    │  Validate        │  │
│  │   URL    │───>│  HTML    │───>│  Content │───>│  Content         │  │
│  └──────────┘    └──────────┘    └──────────┘    └────────┬─────────┘  │
│                                                            │            │
│       ┌────────────────────────────────────────────────────┘            │
│       │                                                                  │
│       v                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │  Chunk   │    │ Generate │    │  Create  │    │  Store in        │  │
│  │  Text    │───>│ Embed-   │───>│ Metadata │───>│  ChromaDB        │  │
│  │          │    │  dings   │    │          │    │                  │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────────────┘  │
│                                                                          │
│       Parameters:                                                        │
│       - Chunk Size: 500 words                                           │
│       - Overlap: 50 words                                               │
│       - Batch Size: 32 (local) / 100 (OpenAI)                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 6.1.1 Content Extraction Strategy

```python
def scrape_url(url: str) -> dict:
    # Primary: Trafilatura (optimized for article extraction)
    text = trafilatura.extract(
        downloaded,
        include_comments=False,
        include_tables=True
    )

    # Fallback: BeautifulSoup (generic HTML parsing)
    if not text:
        soup = BeautifulSoup(response.content, 'html.parser')
        for script in soup(["script", "style"]):
            script.decompose()
        text = soup.get_text(separator='\n', strip=True)

    return {'text': text, 'title': title}
```

#### 6.1.2 Chunking Algorithm

```python
def chunk_text(text: str, chunk_size=500, overlap=50) -> List[str]:
    """
    Overlapping window chunking strategy.

    Visual representation:

    Document: [W1 W2 W3 ... W500 W501 ... W550 W551 ... W1000 ...]
                                |
              +----------------+|+----------------+
              |                 ||                |
              v                 vv                v
    Chunk 1: [W1 --------- W500]
    Chunk 2:          [W451 --------- W950]   (50 word overlap)
    Chunk 3:                    [W901 --------- W1400]

    Benefits:
    - Context continuity across chunk boundaries
    - No information loss at split points
    - Better semantic coherence
    """
```

### 6.2 Query Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         QUERY PROCESSING PIPELINE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────┐                                                           │
│  │  Query   │                                                           │
│  │  Input   │                                                           │
│  └────┬─────┘                                                           │
│       │                                                                  │
│       v                                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    L1 CACHE CHECK (Full Query)                    │  │
│  │  Key: MD5(question + top_k)    TTL: 3600s    Max: 200 entries    │  │
│  └───────────────────────────┬──────────────────────────────────────┘  │
│                     HIT      │      MISS                               │
│              ┌───────────────┴───────────────┐                         │
│              │                               │                          │
│              v                               v                          │
│  ┌──────────────────┐           ┌──────────────────────────────────┐  │
│  │  Return Cached   │           │  L2 CACHE CHECK (Embedding)      │  │
│  │  Result (<5ms)   │           │  Key: MD5(question)              │  │
│  └──────────────────┘           └───────────────┬──────────────────┘  │
│                                        HIT      │      MISS           │
│                                 ┌───────────────┴───────────────┐     │
│                                 │                               │      │
│                                 v                               v      │
│                    ┌──────────────────┐         ┌──────────────────┐  │
│                    │  Use Cached      │         │  Generate New    │  │
│                    │  Embedding       │         │  Embedding       │  │
│                    └────────┬─────────┘         └────────┬─────────┘  │
│                             │                            │            │
│                             └──────────┬─────────────────┘            │
│                                        │                              │
│                                        v                              │
│                           ┌──────────────────────┐                    │
│                           │   ChromaDB Search    │                    │
│                           │   (top_k results)    │                    │
│                           └──────────┬───────────┘                    │
│                                      │                                │
│                                      v                                │
│                           ┌──────────────────────┐                    │
│                           │   Build Context      │                    │
│                           │   (600 char/chunk)   │                    │
│                           └──────────┬───────────┘                    │
│                                      │                                │
│                 ┌────────────────────┼────────────────────┐           │
│                 │                    │                    │           │
│                 v                    v                    v           │
│          ┌──────────┐        ┌──────────────┐     ┌──────────┐       │
│          │ FAST     │        │ L3 CACHE     │     │ LLM      │       │
│          │ MODE     │        │ CHECK        │     │ GENERATE │       │
│          │ (no LLM) │        │ (context)    │     │          │       │
│          └──────────┘        └──────────────┘     └──────────┘       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Prompt Engineering

```python
# Ultra-concise prompt for efficiency
prompt = f"""Answer briefly based on context below.

{context}

Q: {request.question}
A:"""
```

**Design Rationale:**
- Minimal token usage reduces latency and cost
- Direct instruction format improves response quality
- Context-first layout ensures relevant information is prioritized

---

## 7. LLM Provider Abstraction

### 7.1 Provider Interface

Both providers implement a common implicit interface:

```python
class LLMProviderInterface(Protocol):
    """Implicit interface for LLM providers"""

    async def generate_embeddings(self, texts: List[str]) -> List[List[float]]:
        """Generate embeddings for multiple texts"""
        ...

    async def get_cached_embedding(self, text: str) -> List[float]:
        """Get single embedding with caching"""
        ...

    async def generate_chat_completion(
        self,
        prompt: str,
        stream: bool = False,
        max_tokens: int = 500,
        temperature: float = 0.7
    ) -> Union[str, AsyncGenerator]:
        """Generate chat completion"""
        ...

    async def stream_chat_completion(self, prompt: str) -> AsyncGenerator:
        """Stream chat completion responses"""
        ...

    def clear_cache(self) -> None:
        """Clear provider's internal cache"""
        ...

    def get_cache_stats(self) -> dict:
        """Get cache statistics"""
        ...

    def get_provider_info(self) -> dict:
        """Get provider metadata"""
        ...
```

### 7.2 Provider Selection

```python
# Provider factory pattern in main.py
LLM_PROVIDER = os.getenv("LLM_PROVIDER", "openai").lower()

if LLM_PROVIDER == "openai":
    from llm_provider_openai import OpenAIProvider
    llm_provider = OpenAIProvider()
elif LLM_PROVIDER == "local":
    from llm_provider_local import LocalProvider
    llm_provider = LocalProvider()
else:
    raise ValueError(f"Invalid LLM_PROVIDER: {LLM_PROVIDER}")
```

### 7.3 Provider Comparison

| Aspect | OpenAI Provider | Local Provider |
|--------|-----------------|----------------|
| **Setup Complexity** | Simple (API key only) | Moderate (Ollama + models) |
| **Hardware Requirements** | Any CPU | GPU recommended (8GB+ VRAM) |
| **Cost Model** | Pay-per-use (~$1-5/month) | Fixed (electricity) |
| **Privacy** | Data sent to OpenAI | Fully local |
| **Offline Capability** | No | Yes |
| **Embedding Dimensions** | 1536 | 384 |
| **First Query Latency** | 2-4s | 3-8s (GPU) / 10-20s (CPU) |
| **Scaling** | Automatic | Manual |

### 7.4 Model Configuration

```env
# OpenAI Configuration
OPENAI_API_KEY=sk-your-api-key
OPENAI_EMBEDDING_MODEL=text-embedding-3-small  # or text-embedding-3-large
OPENAI_CHAT_MODEL=gpt-4o-mini                  # or gpt-4o, gpt-3.5-turbo

# Local Configuration
LOCAL_EMBEDDING_MODEL=all-MiniLM-L6-v2         # or all-mpnet-base-v2
LOCAL_CHAT_MODEL=llama3.2                      # or phi, mistral, codellama
```

---

## 8. Caching Architecture

### 8.1 Multi-Tier Cache Design

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          MULTI-TIER CACHING                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                         L1 CACHE (Query Results)                     │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │  Key: MD5(question + top_k)                                  │   │ │
│  │  │  Value: Full query response (answer, sources, scores)        │   │ │
│  │  │  TTL: 3600 seconds (1 hour)                                  │   │ │
│  │  │  Max Size: 200 entries                                       │   │ │
│  │  │  Hit Time: <5ms (3000x faster than cold query)               │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                    │                                      │
│                                MISS│                                      │
│                                    v                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                        L2 CACHE (Embeddings)                         │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │  Key: MD5(question)                                          │   │ │
│  │  │  Value: Embedding vector (384 or 1536 dimensions)            │   │ │
│  │  │  Max Size: 200 entries (application) + 500 (provider)        │   │ │
│  │  │  Benefit: Reuse embeddings across different top_k values     │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                    │                                      │
│                                MISS│                                      │
│                                    v                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                      L3 CACHE (LLM Responses)                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │  Key: MD5(context) + question[:50]                           │   │ │
│  │  │  Value: LLM-generated answer                                 │   │ │
│  │  │  Max Size: 200 entries                                       │   │ │
│  │  │  Benefit: Reuse answers for similar context                  │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Cache Management

```python
def manage_cache():
    """LRU-like cache management with TTL"""
    # 1. Remove expired entries
    clean_expired_cache()

    # 2. Evict oldest 20% when over capacity
    for cache_dict in [query_cache, embedding_cache, llm_cache]:
        if len(cache_dict) > CACHE_MAX_SIZE:
            items_to_remove = len(cache_dict) // 5
            oldest_keys = sorted(
                cache_dict.keys(),
                key=lambda k: cache_dict[k].get('timestamp', '')
            )[:items_to_remove]
            for key in oldest_keys:
                del cache_dict[key]
```

### 8.3 Performance Impact

| Scenario | Latency | Cache Level |
|----------|---------|-------------|
| Cold query (no cache) | 2-8 seconds | None |
| L1 hit (full result) | <5ms | L1 Query |
| L2 hit (embedding reuse) | 100-200ms | L2 Embedding |
| L3 hit (LLM response reuse) | 50-100ms | L3 LLM |
| Fast mode (no LLM) | 100-200ms | L2 + Vector Search |

---

## 9. API Design

### 9.1 RESTful API Endpoints

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              API ENDPOINTS                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  SYSTEM                                                                   │
│  ├── GET  /              → System status and provider info                │
│  └── GET  /stats         → System-wide statistics                         │
│                                                                           │
│  URL MANAGEMENT                                                           │
│  ├── POST   /urls/add    → Add URL to track                              │
│  ├── GET    /urls        → List all tracked URLs                         │
│  ├── DELETE /urls/remove → Remove URL from tracking                       │
│  └── POST   /urls/refresh → Manually trigger URL re-crawl                 │
│                                                                           │
│  QUERY                                                                    │
│  ├── POST   /query       → Standard RAG query                            │
│  └── POST   /query/stream → Streaming RAG query (SSE)                    │
│                                                                           │
│  CACHE                                                                    │
│  ├── GET    /cache/stats → Cache statistics                              │
│  └── POST   /cache/clear → Clear all caches                              │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Request/Response Models

#### Query Request

```json
{
  "question": "What is RAG?",
  "top_k": 3,
  "use_cache": true,
  "fast_mode": false
}
```

#### Query Response

```json
{
  "answer": "RAG (Retrieval-Augmented Generation) is...",
  "sources": [
    {
      "url": "https://example.com/rag-guide",
      "title": "Complete RAG Guide"
    }
  ],
  "context_used": ["chunk1...", "chunk2..."],
  "relevance_scores": [0.85, 0.72],
  "cached": false,
  "cache_level": null,
  "fast_mode": false,
  "performance": {
    "total_ms": 2450.5,
    "embedding_ms": 150.2,
    "search_ms": 45.3,
    "llm_ms": 2255.0
  }
}
```

#### Streaming Response (SSE)

```
data: {"sources": [{"url": "...", "title": "..."}]}

data: {"chunk": "RAG"}

data: {"chunk": " stands"}

data: {"chunk": " for..."}

data: {"done": true}
```

### 9.3 Error Handling

```python
# HTTP Status Codes Used
200 OK                 # Successful operation
400 Bad Request        # Invalid input (e.g., duplicate URL)
404 Not Found          # Resource not found (e.g., URL not tracked)
500 Internal Error     # Server-side errors (logged)
```

---

## 10. Frontend Architecture

### 10.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        REACT COMPONENT TREE                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  RAGApplication (Root)                                                    │
│  │                                                                        │
│  ├── Header                                                               │
│  │   ├── Title + Logo                                                     │
│  │   └── Description                                                      │
│  │                                                                        │
│  ├── StatsCards (conditional: stats !== null)                             │
│  │   ├── TotalURLsCard                                                    │
│  │   ├── TotalChunksCard                                                  │
│  │   └── ActiveSourcesCard                                                │
│  │                                                                        │
│  ├── TabContainer                                                         │
│  │   ├── TabNavigation                                                    │
│  │   │   ├── QueryTab                                                     │
│  │   │   └── ManageTab                                                    │
│  │   │                                                                    │
│  │   └── TabContent                                                       │
│  │       │                                                                │
│  │       ├── QueryInterface (activeTab === 'query')                       │
│  │       │   ├── SearchForm                                               │
│  │       │   │   ├── TextInput                                            │
│  │       │   │   └── SearchButton (with loading state)                    │
│  │       │   │                                                            │
│  │       │   └── AnswerDisplay (conditional: answer !== null)             │
│  │       │       ├── AnswerCard                                           │
│  │       │       └── SourcesCard                                          │
│  │       │           ├── RelevanceLegend                                  │
│  │       │           ├── SourceItem[]                                     │
│  │       │           │   ├── RankIndicator                                │
│  │       │           │   ├── RelevanceBadge                               │
│  │       │           │   ├── SourceContent                                │
│  │       │           │   └── RelevanceBar                                 │
│  │       │           └── AverageScoreSummary                              │
│  │       │                                                                │
│  │       └── URLManager (activeTab === 'manage')                          │
│  │           ├── AddURLForm                                               │
│  │           │   ├── URLInput                                             │
│  │           │   ├── IntervalInput                                        │
│  │           │   └── AddButton                                            │
│  │           │                                                            │
│  │           └── URLList                                                  │
│  │               └── URLItem[]                                            │
│  │                   ├── URLLink                                          │
│  │                   ├── StatusBadge                                      │
│  │                   ├── ChunksCount                                      │
│  │                   ├── LastCrawled                                      │
│  │                   ├── RefreshButton                                    │
│  │                   └── DeleteButton                                     │
│  │                                                                        │
│  └── Footer                                                               │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 10.2 State Management

```javascript
// Application State
const [urls, setUrls] = useState([]);           // Tracked URLs array
const [newUrl, setNewUrl] = useState('');       // Add URL form input
const [scanInterval, setScanInterval] = useState(60);  // Crawl interval
const [query, setQuery] = useState('');         // Search query input
const [answer, setAnswer] = useState(null);     // Query response
const [loading, setLoading] = useState(false);  // Loading state
const [stats, setStats] = useState(null);       // System statistics
const [activeTab, setActiveTab] = useState('query');  // Current tab

// Auto-refresh Effect
useEffect(() => {
  fetchUrls();
  fetchStats();
  const interval = setInterval(fetchUrls, 30000);  // 30s refresh
  return () => clearInterval(interval);
}, []);
```

### 10.3 Styling Architecture

```
TailwindCSS Configuration
├── Utility Classes        → Primary styling approach
├── Gradient Backgrounds   → from-blue-50 via-white to-purple-50
├── Responsive Breakpoints → md: prefix for tablet+
├── Animations             → transition-all duration-500
└── Component Patterns
    ├── Cards              → bg-white rounded-lg shadow-sm border
    ├── Buttons            → px-6 py-3 rounded-lg font-medium
    ├── Inputs             → px-4 py-3 border rounded-lg focus:ring-2
    └── Status Badges      → px-2 py-1 text-xs rounded-full
```

### 10.4 API Integration

```javascript
// Base URL Configuration
const API_BASE = process.env.REACT_APP_API_BASE_URL ||
  (process.env.NODE_ENV === 'production'
    ? '/api/v1'
    : 'http://localhost:8000');

// Fetch Pattern
const fetchUrls = async () => {
  try {
    const response = await fetch(`${API_BASE}/urls`);
    const data = await response.json();
    setUrls(data);
  } catch (error) {
    console.error('Error fetching URLs:', error);
  }
};
```

---

## 11. Deployment Architecture

### 11.1 Production Infrastructure

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        PRODUCTION DEPLOYMENT                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                          INTERNET                                   │  │
│  └─────────────────────────────┬──────────────────────────────────────┘  │
│                                │                                          │
│                                │ HTTPS (443)                              │
│                                v                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     Ubuntu 22.04 Server                             │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │                         NGINX                                 │  │  │
│  │  │  - SSL Termination (Let's Encrypt)                           │  │  │
│  │  │  - Static File Serving (React build)                         │  │  │
│  │  │  - Reverse Proxy (/api/v1/* → localhost:8000)                │  │  │
│  │  │  - Security Headers (HSTS, X-Frame-Options, etc.)            │  │  │
│  │  │  - Connection Pooling (keepalive 64)                         │  │  │
│  │  └──────────────────────────┬───────────────────────────────────┘  │  │
│  │                              │                                      │  │
│  │              ┌───────────────┴───────────────┐                      │  │
│  │              │                               │                      │  │
│  │              v                               v                      │  │
│  │  ┌──────────────────────┐      ┌──────────────────────────────┐   │  │
│  │  │   Static Files       │      │   FastAPI Backend            │   │  │
│  │  │   /frontend/build    │      │   (Systemd Service)          │   │  │
│  │  │                      │      │                              │   │  │
│  │  │   - index.html       │      │   - Uvicorn on :8000        │   │  │
│  │  │   - static/js/*      │      │   - Auto-restart on failure │   │  │
│  │  │   - static/css/*     │      │   - Journal logging         │   │  │
│  │  └──────────────────────┘      └──────────────┬───────────────┘   │  │
│  │                                               │                    │  │
│  │                              ┌────────────────┼────────────────┐   │  │
│  │                              │                │                │   │  │
│  │                              v                v                v   │  │
│  │                    ┌──────────────┐  ┌──────────────┐  ┌───────┐  │  │
│  │                    │   SQLite     │  │  ChromaDB    │  │ LLM   │  │  │
│  │                    │   Database   │  │  Vector DB   │  │Provider│  │  │
│  │                    └──────────────┘  └──────────────┘  └───────┘  │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Nginx Configuration

```nginx
# Upstream definition
upstream backend_api {
    server 127.0.0.1:8000;
    keepalive 64;
}

# API Proxy
location /api/v1/ {
    rewrite ^/api/v1/(.*) /$1 break;
    proxy_pass http://backend_api;
    proxy_http_version 1.1;

    # Headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket support for streaming
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    # Disable buffering for streaming
    proxy_buffering off;
}

# Static files with caching
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

### 11.3 Systemd Service

```ini
[Unit]
Description=Konetou RAG Backend API
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/www/konetou/rmad-konetou-v2/rag-application/backend
Environment="PATH=/var/www/konetou/.../venv/bin"
ExecStart=/var/www/konetou/.../venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=10

# Security
NoNewPrivileges=true
PrivateTmp=true

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=konetou-backend

[Install]
WantedBy=multi-user.target
```

### 11.4 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main, master]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Deploy to Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/konetou/rmad-konetou-v2
            git pull origin main
            bash deploy.sh
```

### 11.5 Deployment Script Flow

```
deploy.sh Execution Flow
│
├── 1. Navigate to project directory
│
├── 2. Backend Deployment
│   ├── Activate virtual environment
│   ├── pip install -r requirements.txt
│   ├── Check .env exists
│   └── systemctl restart konetou-backend
│
├── 3. Frontend Deployment
│   ├── npm ci (clean install)
│   ├── npm run build
│   └── chown -R www-data:www-data build
│
├── 4. Nginx Reload
│   ├── nginx -t (test config)
│   └── systemctl reload nginx
│
└── 5. Verification
    ├── Check service status
    └── Test API endpoint
```

---

## 12. Security Architecture

### 12.1 Current Security Measures

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SECURITY LAYERS                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  TRANSPORT LAYER                                                          │
│  ├── SSL/TLS (Let's Encrypt)                                             │
│  ├── HTTPS redirect                                                       │
│  └── HSTS header (max-age=31536000)                                      │
│                                                                           │
│  APPLICATION LAYER                                                        │
│  ├── CORS Configuration (currently: *, should be restricted)             │
│  ├── Input validation (Pydantic models)                                  │
│  └── Error logging (no stack traces in responses)                        │
│                                                                           │
│  INFRASTRUCTURE LAYER                                                     │
│  ├── Nginx security headers                                              │
│  │   ├── X-Frame-Options: SAMEORIGIN                                     │
│  │   ├── X-Content-Type-Options: nosniff                                 │
│  │   └── X-XSS-Protection: 1; mode=block                                 │
│  ├── UFW Firewall (22, 80, 443 only)                                     │
│  └── Systemd security (NoNewPrivileges, PrivateTmp)                      │
│                                                                           │
│  DATA LAYER                                                               │
│  ├── .env files excluded from git                                        │
│  ├── API keys in environment variables                                   │
│  └── SQLite file permissions                                             │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 12.2 Security Considerations

**Current Status:**
- No authentication/authorization implemented
- CORS is wide-open (*)
- No rate limiting
- No request validation beyond Pydantic

**Recommended Enhancements:**
1. API Key Authentication
2. JWT tokens for user sessions
3. Rate limiting per IP
4. CORS restriction to specific origins
5. Input sanitization for URLs
6. SQL injection prevention (parameterized queries - already implemented)

---

## 13. Performance Optimizations

### 13.1 Optimization Summary

| # | Optimization | Impact | Location |
|---|--------------|--------|----------|
| 1 | Multi-tier caching (L1/L2/L3) | 3000x faster cached queries | `main.py` |
| 2 | Connection pooling | Reduced HTTP overhead | `main.py:188-191` |
| 3 | Batch embedding generation | Fewer API calls | Providers |
| 4 | Context truncation (600 chars) | Faster LLM inference | `main.py:520` |
| 5 | Fast mode (skip LLM) | 100-200ms responses | `main.py:531-546` |
| 6 | Async/await pattern | Non-blocking I/O | Throughout |
| 7 | Thread pool for blocking ops | Parallel processing | Local provider |
| 8 | Model pre-warming | Faster first query | Local provider |
| 9 | LRU cache eviction | Bounded memory usage | Cache management |
| 10 | Static asset caching (1 year) | Reduced bandwidth | Nginx |

### 13.2 Expected Performance

| Scenario | OpenAI | Local (GPU) | Local (CPU) |
|----------|--------|-------------|-------------|
| First query | 2-4s | 3-8s | 10-20s |
| Cached query (L1) | <5ms | <5ms | <5ms |
| Fast mode | 100ms | 150ms | 200ms |
| Streaming (first token) | <100ms | <100ms | 200ms |

### 13.3 Bottleneck Analysis

```
Query Latency Breakdown (Cold Query)
│
├── Embedding Generation: 100-500ms
│   └── Optimization: L2 cache, batch processing
│
├── Vector Search: 10-50ms
│   └── Optimization: ChromaDB indexing
│
├── LLM Inference: 2000-6000ms (dominant)
│   └── Optimization: L3 cache, fast mode, streaming
│
└── Network/Serialization: 10-50ms
    └── Optimization: Connection pooling, JSON optimization
```

---

## 14. Scalability Considerations

### 14.1 Current Limitations

| Component | Current Limit | Scaling Path |
|-----------|---------------|--------------|
| SQLite | ~10K URLs | → PostgreSQL |
| In-memory cache | ~200 entries/process | → Redis cluster |
| Single process | 1 CPU core | → Multiple workers |
| Single server | 12GB RAM | → Horizontal scaling |
| ChromaDB | ~1M vectors | → Pinecone/Weaviate |

### 14.2 Recommended Scaling Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      SCALED ARCHITECTURE                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│                        ┌──────────────────┐                              │
│                        │   Load Balancer  │                              │
│                        └────────┬─────────┘                              │
│                                 │                                         │
│         ┌───────────────────────┼───────────────────────┐                │
│         │                       │                       │                │
│         v                       v                       v                │
│  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐         │
│  │  API Node 1  │       │  API Node 2  │       │  API Node N  │         │
│  └──────┬───────┘       └──────┬───────┘       └──────┬───────┘         │
│         │                       │                       │                │
│         └───────────────────────┼───────────────────────┘                │
│                                 │                                         │
│         ┌───────────────────────┼───────────────────────┐                │
│         │                       │                       │                │
│         v                       v                       v                │
│  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐         │
│  │    Redis     │       │  PostgreSQL  │       │   Pinecone   │         │
│  │   (Cache)    │       │   (URLs)     │       │  (Vectors)   │         │
│  └──────────────┘       └──────────────┘       └──────────────┘         │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 14.3 Scaling Recommendations

**Phase 1 (100-1K users):**
- Add Redis for distributed caching
- Implement connection pooling
- Add rate limiting

**Phase 2 (1K-10K users):**
- Migrate to PostgreSQL
- Deploy multiple API workers (Gunicorn)
- Add CDN for static assets

**Phase 3 (10K+ users):**
- Migrate to managed vector DB (Pinecone)
- Kubernetes deployment
- Global load balancing
- Async task queue (Celery)

---

## 15. Appendix

### 15.1 Environment Variables

```env
# Provider Selection
LLM_PROVIDER=openai              # "openai" or "local"

# OpenAI Configuration
OPENAI_API_KEY=sk-...            # Required for OpenAI
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
OPENAI_CHAT_MODEL=gpt-4o-mini

# Local Configuration
LOCAL_EMBEDDING_MODEL=all-MiniLM-L6-v2
LOCAL_CHAT_MODEL=llama3.2
```

### 15.2 Dependencies

**Backend (requirements.txt):**
```
fastapi==0.104.1
uvicorn[standard]==0.24.0
chromadb==0.4.18
python-dotenv==1.0.0
requests==2.31.0
beautifulsoup4==4.12.2
trafilatura==1.6.2
apscheduler==3.10.4
pydantic==2.5.0
openai>=1.0.0
sentence-transformers>=2.7.0
numpy<2.0
ollama==0.1.6
```

**Frontend (package.json):**
```json
{
  "react": "^19.2.0",
  "lucide-react": "^0.552.0",
  "tailwindcss": "^3.4.18"
}
```

### 15.3 API Quick Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | System status |
| GET | `/stats` | System statistics |
| POST | `/urls/add` | Add URL to track |
| GET | `/urls` | List tracked URLs |
| DELETE | `/urls/remove?url=` | Remove URL |
| POST | `/urls/refresh?url=` | Refresh URL |
| POST | `/query` | RAG query |
| POST | `/query/stream` | Streaming query |
| GET | `/cache/stats` | Cache statistics |
| POST | `/cache/clear` | Clear caches |

### 15.4 Performance Testing

```bash
# Run performance tests
./test_performance.sh

# Test scenarios:
# 1. Cold query (no cache)
# 2. Warm query (L1 cache)
# 3. Similar query (L2 cache)
# 4. Fast mode
# 5. Variable top_k
# 6. Streaming
```

### 15.5 Monitoring Commands

```bash
# Backend logs
sudo journalctl -u konetou-backend -f

# Nginx logs
sudo tail -f /var/log/nginx/konetou.error.log

# Service status
sudo systemctl status konetou-backend

# Resource usage
htop
df -h
```

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-12-11 | Claude | Initial comprehensive architecture document |

---

**Konetou by R-Mad Ltd** - Intelligent RAG System for Modern Applications
