# Personal Assistant Implementation Guide

A complete guide to building your own privacy-focused personal assistant with RAG (Retrieval Augmented Generation).

## What You'll Build

A self-hosted AI assistant that:
- Learns from your documents, emails, and notes
- Answers questions using your personal knowledge base
- Helps draft emails and messages
- Runs entirely on your infrastructure (no data sent to AI companies)
- Accessible to family members with authentication

**Example queries:**
- "What was in my email from work about the budget?"
- "Draft a reply to Bob thanking him for the meeting"
- "Summarise the three PDFs I uploaded about the project"

## Architecture

```
Your Documents → Qdrant (Vector DB) → Context
                                        ↓
Your Query → FastAPI → Ollama (Local LLM) → Response
```

**How it works:**
1. Upload documents (PDF, DOCX, emails, etc.)
2. System chunks text and creates embeddings
3. Embeddings stored in Qdrant vector database
4. When you ask a question:
   - Query is embedded
   - Qdrant finds 5 most relevant chunks
   - Context + query sent to Ollama (local LLM)
   - LLM generates answer with sources

## Prerequisites

**Infrastructure:**
- VPS with 8GB RAM minimum (e.g., Hetzner CPX31 £11/month, Fasthosts £20-25/month)
- Ubuntu 22.04 LTS or similar
- SSH access

**Skills:**
- Basic command line familiarity
- Can edit YAML and Python files
- Comfortable with Docker basics (or willing to learn)

**Tools on your local machine:**
- SSH client
- Git
- Text editor

## Cost Breakdown

- **VPS**: £11-25/month (Hetzner or Fasthosts, 8GB RAM)
- **Domain** (optional): £1-2/month
- **Total**: £11-27/month

**No other costs** - all software is free and open source.

## Implementation Steps

### Step 1: Provision VPS (15 minutes)

Choose independent provider (avoid Google/Amazon/Microsoft):
- **Hetzner CPX31**: €13/month (~£11), 8GB RAM, Germany/Finland
- **Fasthosts**: £20-25/month, 8GB RAM, UK

**Setup VPS:**
```bash
# SSH into your VPS
ssh root@your-vps-ip

# Update system
apt update && apt upgrade -y

# Install essentials
apt install -y curl git htop ufw

# Install Docker
curl -fsSL https://get.docker.com | sh

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Verify
docker --version
docker-compose --version

# Create non-root user
adduser llmadmin
usermod -aG sudo,docker llmadmin
```

### Step 2: Create Project Structure (10 minutes)

**On your VPS as llmadmin:**
```bash
mkdir -p ~/pa/{config,src/api,src/rag,src/processors}
cd ~/pa
```

**Create docker-compose.yml:**
```yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: pa-ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    restart: unless-stopped

  qdrant:
    image: qdrant/qdrant:latest
    container_name: pa-qdrant
    ports:
      - "6333:6333"
    volumes:
      - qdrant-data:/qdrant/storage
    restart: unless-stopped

  api:
    build: .
    container_name: pa-api
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src
      - ./config:/app/config
      - uploads:/app/uploads
    environment:
      - OLLAMA_URL=http://ollama:11434
      - QDRANT_URL=http://qdrant:6333
    depends_on:
      - ollama
      - qdrant
    restart: unless-stopped

volumes:
  ollama-data:
  qdrant-data:
  uploads:
```

**Create Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src /app/src

EXPOSE 8000

CMD ["uvicorn", "src.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Create requirements.txt:**
```
fastapi==0.109.0
uvicorn[standard]==0.27.0
pydantic==2.6.0
httpx==0.26.0
sentence-transformers==2.3.1
qdrant-client==1.7.3
python-multipart==0.0.6
pypdf==4.0.1
python-docx==1.1.0
aiofiles==23.2.1
pyyaml==6.0.1
```

### Step 3: Implement FastAPI Application (30 minutes)

**Create src/api/main.py:**
```python
from fastapi import FastAPI, UploadFile, File, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import httpx
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer

app = FastAPI(title="Personal Assistant API")

# Initialize clients
qdrant = QdrantClient(url="http://qdrant:6333")
embedder = SentenceTransformer('all-MiniLM-L6-v2')

class ChatRequest(BaseModel):
    message: str
    user_id: str = "default"

class ChatResponse(BaseModel):
    response: str
    sources: List[dict] = []

@app.get("/health")
async def health():
    return {"status": "healthy"}

@app.post("/upload")
async def upload_document(file: UploadFile = File(...), user_id: str = "default"):
    """Upload and process document"""
    # TODO: Implement document processing
    # 1. Save file
    # 2. Extract text (use pypdf, python-docx)
    # 3. Chunk text (512 tokens, 50 overlap)
    # 4. Generate embeddings
    # 5. Store in Qdrant with user_id filter

    return {"status": "uploaded", "filename": file.filename}

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """Chat with RAG"""
    # 1. Embed query
    query_vector = embedder.encode(request.message).tolist()

    # 2. Search Qdrant for relevant chunks
    search_results = qdrant.search(
        collection_name="documents",
        query_vector=query_vector,
        limit=5,
        query_filter={"must": [{"key": "user_id", "match": {"value": request.user_id}}]}
    )

    # 3. Build context from results
    context = "\n\n".join([hit.payload["text"] for hit in search_results])

    # 4. Build prompt
    prompt = f"""Context from user's documents:
{context}

User question: {request.message}

Answer the question using the context above. If the answer isn't in the context, say so."""

    # 5. Call Ollama
    async with httpx.AsyncClient(timeout=60.0) as client:
        response = await client.post(
            "http://ollama:11434/api/generate",
            json={"model": "qwen2.5:7b", "prompt": prompt, "stream": False}
        )

    result = response.json()

    return {
        "response": result["response"],
        "sources": [{"text": hit.payload["text"][:200]} for hit in search_results]
    }
```

**Note:** This is a **minimal working example**. You'll need to:
- Implement document processing (PDF, DOCX extraction)
- Add proper chunking logic
- Create Qdrant collection
- Add error handling
- Implement authentication

### Step 4: Start Services (5 minutes)

```bash
cd ~/pa

# Start everything
docker-compose up -d

# Pull LLM model (takes 10-15 minutes, ~4GB download)
docker exec pa-ollama ollama pull qwen2.5:7b

# Check services
docker-compose ps

# View logs
docker-compose logs -f api
```

### Step 5: Test Basic Functionality (5 minutes)

```bash
# Test health
curl http://localhost:8000/health

# Test Ollama
docker exec pa-ollama ollama list

# Test Qdrant
curl http://localhost:6333/healthz
```

### Step 6: Add Authentication (20 minutes)

**Install nginx and setup basic auth:**
```bash
# On VPS
sudo apt install -y nginx apache2-utils

# Create password file (run for each family member)
sudo htpasswd -c /etc/nginx/.htpasswd you
sudo htpasswd /etc/nginx/.htpasswd partner
sudo htpasswd /etc/nginx/.htpasswd child

# Create nginx config: /etc/nginx/sites-available/pa
sudo nano /etc/nginx/sites-available/pa
```

**Nginx configuration:**
```nginx
server {
    listen 80;
    server_name _;

    auth_basic "Family Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_buffering off;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/pa /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx

# Test with auth
curl -u you:password http://your-vps-ip/health
```

### Step 7: Setup Remote Access (20 minutes)

**Option A: WireGuard (Most Private)**
```bash
# On VPS
sudo apt install -y wireguard

# Generate keys
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key

# Configure /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <server_private.key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_public.key>
AllowedIPs = 10.0.0.2/32

# Start WireGuard
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Open firewall
sudo ufw allow 51820/udp
```

**On your laptop**, install WireGuard and configure client with server details. Access PA at `http://10.0.0.1`.

**Option B: Tailscale (Easier)**
```bash
# On VPS
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# On laptop: install Tailscale app, login
# Access PA by hostname
```

## Next Steps

### Implement Document Processing

Add to `src/processors/documents.py`:
- PDF text extraction (pypdf)
- DOCX text extraction (python-docx)
- Text chunking (512 tokens, 50 overlap)
- Metadata extraction (filename, date, etc.)

### Implement RAG Properly

Add to `src/rag/`:
- Proper embedding generation
- Qdrant collection creation
- Smart chunking strategies
- Re-ranking for better results

### Add Features

- Conversation history
- Email integration (IMAP polling)
- Web UI (React/Vue frontend)
- Voice interface (Whisper + TTS)

### Production Hardening

- Add proper logging
- Implement rate limiting
- Set up backups (see migration guide)
- Add monitoring (Prometheus + Grafana)
- SSL certificates (Let's Encrypt)

## Troubleshooting

**Services won't start:**
```bash
docker-compose logs
# Check for port conflicts, permission issues
```

**Out of memory:**
```bash
free -h
docker stats
# May need to upgrade VPS or use smaller model
```

**Ollama model too large:**
```bash
# Use smaller model
docker exec pa-ollama ollama pull qwen2.5:3b
# Update docker-compose.yml to use 3b model
```

**Can't connect remotely:**
```bash
# Check firewall
sudo ufw status
# Check WireGuard/Tailscale status
sudo systemctl status wg-quick@wg0
```

## Model Choices

**LLM Model (choose one):**
- **Qwen2.5 7B** (recommended): Best reasoning, fits 8GB RAM
- **Llama 3.1 8B**: Great conversation, Meta-trained
- **Mistral 7B**: Fastest, most compact

**Embedding Model:**
- **all-MiniLM-L6-v2** (recommended): Fast, 384 dimensions
- **all-mpnet-base-v2**: Better quality, slower, 768 dimensions

## Estimated Timeline

- **Day 1 (2-3 hours)**: VPS setup, Docker, basic services
- **Day 2 (2-3 hours)**: FastAPI implementation, test basic RAG
- **Day 3 (1-2 hours)**: Authentication, remote access
- **Week 2+**: Document processing, features, polish

**Total to working prototype: 6-8 hours over a weekend**

## Cost Comparison

| Setup | Cost/Month | Privacy | Effort |
|-------|-----------|---------|--------|
| ChatGPT Plus | £16 | ❌ Sent to OpenAI | 5 min |
| Self-Hosted | £11-25 | ✅ Stays local | 6-8 hours |

**Break-even:** 3-4 months if you value your time at £50/hour. Immediately if privacy is priority.

## Important Notes

⚠️ **This is a minimal implementation guide.** You'll need to:
- Add proper error handling
- Implement security best practices
- Handle edge cases
- Add monitoring and logging

✅ **But it will work!** This gets you a functional personal assistant you can iterate on.

📚 **For deeper understanding**, read:
- [Architecture guide](../../docs/architecture.md)
- [Privacy guide](../../docs/privacy.md)
- [Migration guide](../../docs/migration.md)

## Getting Help

- Check parent guide documentation
- Review Docker logs: `docker-compose logs`
- Test each component independently
- Start simple, add features incrementally

## Keep Data Private

Create your implementation in a **private Git repository**. Never commit:
- `.env` files with secrets
- `.htpasswd` user credentials
- Uploaded documents
- Personal configuration

**Your data, your infrastructure, your control.** 🔒
