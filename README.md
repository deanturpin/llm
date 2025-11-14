# Self-Hosted LLM Guide

A comprehensive guide to building privacy-focused AI applications using open-source LLMs on your own infrastructure.

## Why Self-Host?

- **Privacy**: Your data never leaves your infrastructure
- **Independence**: Avoid Google, Amazon, and Microsoft infrastructure entirely
- **Cost**: After initial setup, cheaper than API services for heavy use
- **Control**: Choose models, customise behaviour, no rate limits
- **Learning**: Understand how LLMs work under the hood
- **Family/team access**: Run for multiple users on shared infrastructure

## Use Cases Covered

### 1. Personal Assistant ([guide](./use-cases/personal-assistant.md))
Query your documents, draft emails, manage tasks with natural language.

**Tech**: Ollama + Qdrant + FastAPI + RAG
**Cost**: From £11/month (Hetzner VPS)
**Complexity**: Moderate

### 2. Coding Partner ([guide](./use-cases/coding-partner.md))
Claude Code-style agentic assistant that can read/write code, use git, execute commands.

**Tech**: Code-specific LLM + tool framework + RAG over codebase
**Cost**: From £44/month (needs more RAM/CPU)
**Complexity**: Advanced

### 3. Google Photos Alternative ([guide](./use-cases/photo-management.md))
Semantic photo search, face recognition, AI albums - all private.

**Tech**: CLIP + LLaVA + Immich/PhotoPrism + Qdrant
**Cost**: From £50/month (needs GPU or larger VPS)
**Complexity**: Moderate-Advanced

## Quick Start

Choose your use case above and follow the specific guide. All guides include:
- Architecture diagrams
- Docker Compose configurations
- Deployment scripts
- Cost breakdowns
- Security best practices

## Repository Structure

```
llm-guide/
├── use-cases/
│   ├── personal-assistant.md    # Full guide for PA use case
│   ├── coding-partner.md        # Full guide for coding assistant
│   └── photo-management.md      # Full guide for photos
├── implementations/
│   ├── personal-assistant/      # Docker configs, code for PA
│   ├── coding-partner/          # Docker configs, code for coder
│   └── photo-management/        # Docker configs, code for photos
└── docs/
    ├── vps-providers.md         # Independent VPS providers (avoid big tech)
    ├── migration.md             # Moving between providers (fully portable)
    ├── privacy.md               # Privacy and data security explained
    ├── infrastructure.md        # Home server, networking, access methods
    ├── architecture.md          # Technical architecture details
    └── setup.md                 # Deployment walkthrough
```

## General Architecture Pattern

All use cases follow this pattern:

```
┌─────────────────────────────────────┐
│      Cloudflare Tunnel (free)       │
│    Handles: TLS, DDoS protection    │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│         Nginx + Auth Layer          │
└─────────────┬───────────────────────┘
              │
     ┌────────┴────────┐
     │                 │
┌────▼─────┐    ┌─────▼──────┐
│ App/API  │    │  Qdrant    │
│  Layer   │◄───┤ (vectors)  │
└────┬─────┘    └────────────┘
     │
┌────▼─────┐
│  Ollama  │
│  (LLM)   │
└──────────┘
```

## Infrastructure Recommendations

### Budget Option (£11-25/month)
- **VPS**: Hetzner CPX31 (£11) or Fasthosts 8GB (£20-25)
- **Independence**: ⭐⭐⭐⭐⭐ (Own infrastructure, no big tech)
- **Good for**: Personal assistant, light coding help
- **Models**: 7B-8B models (Qwen2.5, Llama 3.1, Mistral)

### Performance Option (£44-90/month)
- **VPS**: Hetzner CCX33 (£44) or Fasthosts 32GB (£70-90)
- **Independence**: ⭐⭐⭐⭐⭐ (Own infrastructure, no big tech)
- **Good for**: All use cases, multiple concurrent users
- **Models**: Up to 14B models, or multiple 7B models

### GPU Option (£80-100/month)
- **VPS + GPU**: Hetzner CCX23 + RunPod spot instance
- **Good for**: Photo management, multimodal, fast inference
- **Models**: 70B models, vision models (LLaVA), CLIP

### Home Server (£500-2000 one-time)
- **Hardware**: Mini PC or custom build with optional GPU
- **Running costs**: £6-30/month (electricity)
- **Good for**: Maximum privacy and control, heavy usage
- **Break-even**: ~2-3 years vs VPS

See guides:
- [VPS providers](./docs/vps-providers.md) - Independent providers (no Google/Amazon/Microsoft)
- [Migration guide](./docs/migration.md) - Moving between providers (30-60 min, fully portable)
- [Infrastructure guide](./docs/infrastructure.md) - Detailed setup and comparisons

## Key Technologies

- **LLM Runtime**: [Ollama](https://ollama.ai/) - Easy model management
- **Vector DB**: [Qdrant](https://qdrant.tech/) - Fast similarity search
- **Web Framework**: [FastAPI](https://fastapi.tiangolo.com/) - Modern Python API
- **Deployment**: Docker Compose - Simple orchestration
- **Access**: [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) - Secure remote access

## Contributing

This guide is open source! Contributions welcome:
- Add new use cases
- Improve deployment scripts
- Share your architecture variations
- Report issues or suggestions

## Privacy & Security

**All implementations in this guide prioritise privacy:**
- Run entirely on your infrastructure (no data sent to AI companies)
- Use local model inference via Ollama (models run offline)
- Support independent VPS providers (FastHosts, OVH, or home server)
- Include authentication and rate limiting
- Support encrypted storage and backups
- Multiple access options (WireGuard, Tailscale, or Cloudflare Tunnel)

**Read more:**
- [Independent VPS providers](./docs/vps-providers.md) - Fasthosts, Hetzner, OVH, and others (no Google/Amazon/Microsoft)
- [Migration guide](./docs/migration.md) - Easily move between providers or to home server
- [Privacy explained](./docs/privacy.md) - How your data stays private with local models
- [Infrastructure options](./docs/infrastructure.md) - Home servers, networking, and access methods

## For Your Implementation

**This is a public guide.** For your actual implementation:
1. Fork/clone this repo
2. Create a **private repository** for your data and config
3. Follow the use-case guides but keep your:
   - API keys and passwords
   - User configurations
   - Uploaded documents
   - Personal data

...in your private repo, not here!

## License

MIT - Use freely, attribution appreciated
