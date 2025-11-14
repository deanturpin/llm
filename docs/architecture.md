# Architecture Design

## Overview

This personal assistant system uses a RAG (Retrieval Augmented Generation) architecture with local LLM deployment for privacy and control.

## Component Details

### 1. Ollama (LLM Runtime)
- **Purpose**: Run open-source LLMs efficiently
- **Models**: Llama 3.1 8B, Qwen2.5 7B, or Mistral 7B
- **Quantisation**: Q4_K_M for 8GB RAM, Q5_K_M for 16GB+ RAM
- **API**: OpenAI-compatible REST API
- **Memory**: ~6GB for 8B model with Q4 quantisation

### 2. Qdrant (Vector Database)
- **Purpose**: Store and retrieve document embeddings
- **Features**:
  - HNSW indexing for fast similarity search
  - Payload filtering (by user, date, source)
  - On-disk storage with optional encryption
- **Memory**: ~1-2GB base + stored vectors
- **Embeddings**: all-MiniLM-L6-v2 (384 dimensions, fast and efficient)

### 3. FastAPI (API Layer)
- **Endpoints**:
  - `POST /chat` - conversational interface
  - `POST /upload` - document upload and processing
  - `GET /search` - semantic search over documents
  - `GET /history` - conversation history
  - `DELETE /documents/{id}` - remove documents
- **Features**:
  - Streaming responses
  - Multi-user context isolation
  - Rate limiting (10 req/min per user)
  - Request logging

### 4. Nginx (Reverse Proxy)
- **Purpose**: Handle authentication and routing
- **Features**:
  - Basic Auth or OAuth2 Proxy
  - TLS termination
  - Request buffering
  - Static file serving (future web UI)

### 5. Cloudflare Tunnel
- **Purpose**: Secure access without exposing ports
- **Benefits**:
  - Free tier sufficient for family use
  - DDoS protection
  - Automatic TLS certificates
  - No need to configure firewall/router

## Data Flow

### Document Upload
```
User uploads document
    ↓
FastAPI receives file
    ↓
Document processor extracts text
    ↓
Text chunked (512 token chunks, 50 token overlap)
    ↓
Embeddings generated (all-MiniLM-L6-v2)
    ↓
Stored in Qdrant with metadata
```

### Chat Query
```
User sends message
    ↓
FastAPI authenticates user
    ↓
Query embedded
    ↓
Qdrant retrieves top-k relevant chunks (k=5)
    ↓
Context + query → LLM prompt
    ↓
Ollama generates response
    ↓
Response streamed to user
```

## Resource Allocation (8GB VPS)

- **Ollama**: 6GB (8B model Q4)
- **Qdrant**: 1GB
- **FastAPI**: 512MB
- **Nginx**: 256MB
- **System**: 256MB
- **Total**: ~8GB

## Scaling Options

### If needing more capacity:

1. **Upgrade VPS**: CCX33 (32GB RAM) allows:
   - Larger models (13B or 70B quantised)
   - More concurrent users
   - Larger document corpus

2. **Add GPU VPS**:
   - Hetzner doesn't offer GPU
   - Consider RunPod (~$0.20/hr spot = ~£35/month for RTX 3090)
   - Allows larger models and faster inference

3. **Hybrid approach**:
   - Keep Qdrant and API on Hetzner
   - Run Ollama on RunPod GPU instance
   - Connect via private network

## Security Architecture

```
Internet
    ↓
Cloudflare Tunnel (TLS, DDoS protection)
    ↓
Nginx (Authentication)
    ↓
FastAPI (User context isolation)
    ↓
Qdrant (Per-user collections)
```

### Authentication Options

**Option A: Basic Auth** (simpler)
- Nginx htpasswd file
- One credential per family member
- Easy to manage

**Option B: OAuth2 Proxy** (more secure)
- Google/GitHub OAuth
- Whitelist family email addresses
- Better audit trail

**Recommendation**: Start with Basic Auth, migrate to OAuth2 if needed.

## Backup Strategy

1. **Qdrant data**: Daily backup to Hetzner Storage Box or Cloudflare R2
2. **Conversation history**: PostgreSQL dump (if implemented)
3. **Configuration**: Git repository (this repo)

## Privacy Considerations

- **All data stays on VPS**: No external API calls for inference
- **Encrypted at rest**: Qdrant supports encryption
- **Encrypted in transit**: TLS via Cloudflare
- **User isolation**: Each family member has separate document collections
- **Audit logging**: Track who accessed what (optional)

## Future Enhancements

1. **Email Integration**: IMAP polling → auto-index emails
2. **Google Drive Sync**: Periodic sync of Drive documents
3. **Multi-modal**: Add image understanding (LLaVA model)
4. **Fine-tuning**: Train on your writing style for better email drafting
5. **Web UI**: React frontend for better UX
6. **Mobile app**: Flutter app for iOS/Android
