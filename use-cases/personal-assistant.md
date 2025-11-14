# Personal Assistant Use Case

A privacy-focused personal assistant that learns from your documents, emails, and notes to help with:
- Information retrieval ("What was my dentist appointment date?")
- Email drafting ("Draft a reply to Jane about the project")
- Document summarisation ("Summarise these three PDFs")
- Task management (natural language TODO tracking)

## Architecture

```
┌──────────────────┐
│  You (anywhere)  │
└────────┬─────────┘
         │ HTTPS
         │
┌────────▼─────────┐
│ Cloudflare Tunnel│ ← Free, handles TLS + DDoS
└────────┬─────────┘
         │
┌────────▼─────────┐
│  Nginx (Auth)    │ ← Basic auth for family members
└────────┬─────────┘
         │
┌────────▼─────────┐     ┌──────────────┐
│  FastAPI         │────►│  Qdrant      │ ← Your documents
│  (Orchestration) │◄────│  (Vector DB) │   embedded here
└────────┬─────────┘     └──────────────┘
         │
┌────────▼─────────┐
│  Ollama          │ ← Local LLM (Qwen2.5 7B)
│  (LLM Runtime)   │
└──────────────────┘
```

## How It Works

### 1. Document Upload
```
You upload document.pdf
    ↓
FastAPI receives it
    ↓
Text extracted (PyPDF, python-docx, etc.)
    ↓
Text chunked (512 tokens, 50 overlap)
    ↓
Each chunk → embedding (all-MiniLM-L6-v2)
    ↓
Stored in Qdrant with metadata (filename, date, user)
```

### 2. Chat Query
```
You ask: "What did Jane say about the budget?"
    ↓
Query → embedding
    ↓
Qdrant finds 5 most relevant chunks from your docs
    ↓
Build prompt: [context from chunks] + [your question]
    ↓
Ollama (Qwen2.5 7B) generates answer
    ↓
Response + source citations returned
```

### 3. RAG Magic

**Without RAG:**
> You: "What's my dentist appointment?"
> LLM: "I don't have access to your personal information."

**With RAG:**
> You: "What's my dentist appointment?"
> LLM: *[Retrieves your calendar email]*
> "Your dentist appointment is Thursday 21st November at 2pm with Dr Smith."
> *Source: email from dentist-office@example.com, received 10 Nov*

## Cost Breakdown

### Budget Setup (£11/month)
- **Hetzner CPX31**: £11/month (4 vCPU, 8GB RAM, 160GB SSD)
- **Cloudflare Tunnel**: Free
- **Domain**: £1-2/month (optional)
- **Total**: £11-13/month

**Suitable for**:
- 1-5 family members
- ~10,000 documents
- Qwen2.5 7B or Llama 3.1 8B models
- Light-moderate usage

### Performance Setup (£44/month)
- **Hetzner CCX33**: £44/month (8 vCPU, 32GB RAM, 240GB SSD)
- **Cloudflare Tunnel**: Free
- **Domain**: £1-2/month (optional)
- **Total**: £44-46/month

**Suitable for**:
- 5-20 users
- Unlimited documents (add storage volume if needed)
- Larger models (Qwen2.5 14B) or multiple models
- Heavy usage, fast responses

## Model Recommendations

### Best All-Rounder: Qwen2.5 7B
- Great at reasoning and instruction following
- Good code understanding (if you paste snippets)
- Fast on 8GB RAM (Q4 quantisation)
- Excellent for UK English

### Best for Conversation: Llama 3.1 8B
- Natural dialogue
- Good context understanding
- Meta-trained, widely supported
- Slightly slower than Qwen

### Best for Speed: Mistral 7B
- Fast inference
- Compact context window
- Good general knowledge
- Less chatty, more direct

**My recommendation**: Start with **Qwen2.5 7B**

## Feature Roadmap

### Phase 1: Core (1-2 weeks to implement)
- ✅ Document upload (PDF, DOCX, TXT, MD)
- ✅ RAG-based chat
- ✅ Basic auth
- ✅ Cloudflare Tunnel deployment

### Phase 2: Enhanced (1-2 weeks)
- Email integration (IMAP polling)
- Conversation history
- Web UI (basic chat interface)
- Google Drive sync

### Phase 3: Advanced (ongoing)
- Voice interface (Whisper + TTS)
- Mobile app
- Fine-tuning on your writing style
- Calendar integration
- Proactive reminders

## Implementation Guide

See [../implementations/personal-assistant/](../implementations/personal-assistant/) for:
- `docker-compose.yml` - Full stack configuration
- `src/` - Python FastAPI application
- `scripts/` - Deployment and setup scripts
- Detailed README with step-by-step instructions

## Quick Deploy

```bash
# Clone this repo
git clone https://github.com/yourusername/llm-guide
cd llm-guide/implementations/personal-assistant

# Set up authentication
./scripts/setup-auth.sh

# Deploy to VPS
./scripts/deploy-pa.sh user@your-vps.example.com

# Set up Cloudflare Tunnel (one-time)
ssh user@your-vps.example.com
cloudflared tunnel login
cloudflared tunnel create pa
# ... follow setup instructions
```

## Sample Queries

Once deployed, you can:

```bash
# Upload a document
curl -u you:password -X POST \
  -F "file=@/path/to/doc.pdf" \
  https://assistant.yourdomain.com/upload

# Ask questions
curl -u you:password -X POST \
  -H "Content-Type: application/json" \
  -d '{"message": "Summarise my recent emails from work", "user_id": "you"}' \
  https://assistant.yourdomain.com/chat

# Search your docs
curl -u you:password \
  "https://assistant.yourdomain.com/search?query=budget%20meeting&user_id=you"
```

## Privacy & Security

✅ **All data stays on your VPS** - no external API calls
✅ **Encrypted in transit** - TLS via Cloudflare
✅ **Authentication required** - per-user credentials
✅ **Rate limited** - 10 requests/minute per user
✅ **User isolation** - each family member has separate document collection
✅ **No logging of sensitive data** - only audit logs (optional)

## Comparison: Self-Hosted vs Cloud Services

| Feature | ChatGPT Plus | Google Gemini | Self-Hosted PA |
|---------|--------------|---------------|----------------|
| **Cost** | £16/month | Free-£15/month | £11-44/month |
| **Privacy** | ❌ Sent to OpenAI | ❌ Sent to Google | ✅ Stays local |
| **Custom data** | Upload per chat | Limited | ✅ Persistent RAG |
| **Family access** | Separate accounts | Separate accounts | ✅ Shared infra |
| **Offline** | ❌ No | ❌ No | ⚠️ VPS needed |
| **Control** | ❌ No | ❌ No | ✅ Full |
| **Setup time** | 5 min | 5 min | 2-4 hours |

## FAQs

**Q: Can I run this on a home server instead of VPS?**
A: Yes! Same Docker setup works. Use Cloudflare Tunnel or Tailscale for remote access.

**Q: How much storage do I need?**
A: Text documents are tiny. 10,000 documents ≈ 5-10GB. Add Hetzner Storage Volume for large media.

**Q: Can I add more family members later?**
A: Yes, just run `htpasswd -B .htpasswd newuser` and restart nginx.

**Q: What happens if the VPS goes down?**
A: Your data is safe (backup regularly). Restart VPS and services auto-start. Set up monitoring (uptime.robot, free tier).

**Q: Can I migrate to a more powerful VPS later?**
A: Yes! Backup Qdrant data and uploads, provision new VPS, restore, redeploy. ~30 min downtime.

**Q: Does this work for non-English languages?**
A: Yes! Qwen2.5 is multilingual. For better results in specific languages, use language-specific embeddings model.
