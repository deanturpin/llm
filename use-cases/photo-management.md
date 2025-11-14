# Google Photos Alternative Use Case

A privacy-focused photo management system with AI-powered features:
- Semantic search ("Find photos of me at the beach")
- Face recognition ("Show all photos with Sarah")
- Auto-tagging and categorisation
- Memories and auto-generated albums
- Multi-user access (family sharing)

## Why Replace Google Photos?

**Google Photos Advantages:**
- Free (15GB) or cheap (100GB for £2/month)
- Excellent AI search and face recognition
- Zero setup, just works
- Mobile apps are polished

**Privacy Concerns:**
- Google scans all your photos
- Trains AI models on your data
- Can be subpoenaed
- Terms of service can change

**Self-Hosted Advantages:**
- Complete privacy and control
- No usage limits or quotas
- Custom AI features
- Family data stays together
- No vendor lock-in

## Architecture

```
┌─────────────────────────────────┐
│   Immich/PhotoPrism Web UI      │ ← Photo browsing, upload
│   (Like Google Photos interface)│
└────────────┬────────────────────┘
             │
    ┌────────┴─────────┐
    │                  │
┌───▼──────┐   ┌──────▼─────┐
│  CLIP    │   │  LLaVA     │  ← AI models
│(Semantic)│   │(Captioning)│
└───┬──────┘   └──────┬─────┘
    │                 │
    └────────┬────────┘
             │
    ┌────────▼─────────┐
    │     Qdrant       │  ← Image embeddings
    │   (Vector DB)    │
    └──────────────────┘
```

## Key Features

### 1. Semantic Search
"Find photos of sunset over water" - finds images matching description, even without tags

**Tech**: CLIP (OpenAI's vision-language model)
- Encodes images and text into same embedding space
- Extremely accurate semantic matching
- No manual tagging required

### 2. Face Recognition
"Show all photos with Dad" - automatically recognises and groups faces

**Tech**: InsightFace or similar
- Detects faces in photos
- Creates face embeddings
- Clusters similar faces
- You label clusters once ("This is Dad")

### 3. Auto-Captioning
Each photo gets descriptive caption: "Two people sitting on a beach at sunset with palm trees"

**Tech**: LLaVA (vision-language model)
- Analyses image content
- Generates natural language description
- Useful for search and accessibility

### 4. Smart Albums
Auto-generate albums: "Beach holidays 2023", "Photos with family"

**Tech**: Clustering + LLM
- Group by embeddings similarity
- Detect events (time + location clustering)
- LLM generates album name and description

## Implementation Options

### Option A: Immich (Recommended)
Open-source Google Photos clone with AI features

**Pros:**
- Beautiful mobile apps (iOS + Android)
- Google Photos-like interface
- Active development
- Built-in ML features
- Docker deployment

**Cons:**
- More opinionated (less customisable)
- Heavier resource usage

### Option B: PhotoPrism
Feature-rich photo management with AI

**Pros:**
- Excellent web UI
- Strong AI features (faces, labels, places)
- Highly customisable
- Good docs

**Cons:**
- No official mobile app
- Some features require subscription

### Option C: Custom Build
Build on Immich/PhotoPrism and add your own AI

**Pros:**
- Full control
- Custom features
- Learn the tech

**Cons:**
- More work
- Maintenance burden

**Recommendation**: Start with **Immich**, add custom AI later if needed

## Resource Requirements

This is **more demanding** than the personal assistant:

### Minimum (CPU-only)
- **VPS**: Hetzner CCX33 (32GB RAM) - £44/month
- **Storage**: 1TB Hetzner Storage Box - £3/month
- **Total**: ~£47/month

**Performance**:
- CLIP inference: ~2-3s per image (CPU)
- Face detection: ~1-2s per face
- Suitable for: Small-medium libraries (<10k photos)

### Recommended (with GPU)
- **VPS**: Hetzner CCX23 (16GB RAM) - £28/month
- **GPU**: RunPod RTX 3090 spot - ~£35/month
- **Storage**: 1TB Storage Box - £3/month
- **Total**: ~£66/month

**Performance**:
- CLIP inference: ~100ms per image
- Face detection: ~200ms per face
- Suitable for: Large libraries (100k+ photos)

### Premium (Dedicated)
- **VPS**: Hetzner CCX63 (64GB RAM) - £173/month with GPU
- Or build home server with GPU
- **Storage**: NAS or large SSD
- **Total**: £173/month or one-time hardware cost

## Cost Comparison

| Service | Storage | Cost/month | Privacy | Features |
|---------|---------|------------|---------|----------|
| **Google Photos** | 100GB | £2 | ❌ Poor | ⭐⭐⭐⭐⭐ |
| **Google Photos** | 2TB | £8 | ❌ Poor | ⭐⭐⭐⭐⭐ |
| **iCloud** | 2TB | £7 | ⚠️ OK | ⭐⭐⭐⭐ |
| **Self-hosted (CPU)** | 1TB | £47 | ✅ Excellent | ⭐⭐⭐⭐ |
| **Self-hosted (GPU)** | 1TB | £66 | ✅ Excellent | ⭐⭐⭐⭐⭐ |

**Break-even analysis**:
- If you value privacy: Self-hosting worth it at any price
- If purely cost: Google Photos cheaper until ~10TB
- If features + privacy: Self-hosted GPU option is sweet spot

## Model Selection

### For Semantic Search: CLIP
- **Model**: `openai/clip-vit-large-patch14`
- **Size**: 1.7GB
- **Speed**: Fast (especially on GPU)
- **Accuracy**: Excellent

### For Captioning: LLaVA or Llama 3.2-Vision
- **Model**: `llava:7b` or `llama3.2-vision:11b`
- **Size**: 4-8GB
- **Speed**: Moderate (needs GPU for comfort)
- **Accuracy**: Very good

### For Faces: InsightFace
- **Model**: `buffalo_l` or `antelopev2`
- **Size**: ~500MB
- **Speed**: Fast
- **Accuracy**: Excellent

## Deployment Architecture

```yaml
# docker-compose.yml
services:
  immich:          # Photo management UI
  postgres:        # Immich database
  redis:           # Immich cache

  ollama:          # LLaVA for captioning

  clip-service:    # CLIP embeddings API
  face-service:    # Face detection API

  qdrant:          # Vector database for search

  nginx:           # Reverse proxy
```

## Storage Strategy

Photos are **large**. Strategy matters:

### Option 1: Object Storage
- Hetzner Storage Box (S3-compatible)
- Cheap: £3/TB/month
- Slower access
- Good for archives

### Option 2: Block Storage
- Hetzner Volumes attached to VPS
- Fast access
- More expensive: £8/100GB/month
- Good for active library

### Option 3: Hybrid
- Recent photos (1 year): Block storage (fast)
- Archive: Object storage (cheap)
- Best of both worlds

### Recommendation
Start with **Hetzner Storage Box** (cheap), add Volume if performance is an issue.

## Feature Comparison

| Feature | Google Photos | Self-Hosted |
|---------|--------------|-------------|
| Semantic search | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Face recognition | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Auto-tagging | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Mobile apps | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ (Immich) |
| Privacy | ⭐ | ⭐⭐⭐⭐⭐ |
| Sharing | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Cost (2TB) | £8/m | £47-66/m |
| Setup complexity | ⭐⭐⭐⭐⭐ | ⭐⭐ |

## Implementation Guide

See [../implementations/photo-management/](../implementations/photo-management/) for:
- Full Docker Compose setup
- CLIP integration scripts
- Face recognition pipeline
- Immich configuration
- Deployment guide

## Quick Deploy

```bash
# Provision VPS with GPU
# Or use CPU-only setup (slower)

cd implementations/photo-management

# Configure storage
./scripts/setup-storage.sh

# Deploy stack
./scripts/deploy-photos.sh user@your-vps.com

# Import existing library
./scripts/import-photos.sh /path/to/google-takeout
```

## Migration from Google Photos

1. **Export from Google Takeout**
   - Request archive at takeout.google.com
   - Download ZIP files (may be split)

2. **Import to self-hosted**
   ```bash
   # Immich has Google Photos import tool
   docker exec immich-cli import-google-photos /path/to/takeout
   ```

3. **Process with AI**
   - CLIP embeddings generated automatically
   - Face detection runs in background
   - Captioning can be batch-processed

4. **Verify and cleanup**
   - Check all photos imported
   - Verify face clusters
   - Label unknown faces
   - Delete Google Photos data (optional)

## Advanced Features

### Duplicate Detection
Find and remove duplicate photos automatically:
- Perceptual hashing (pHash)
- Similar embedding detection
- Visual similarity scoring

### Location Clustering
Group photos by location:
- EXIF GPS data extraction
- Reverse geocoding
- Event detection (same place + time)

### Quality Scoring
Auto-detect best photos:
- Blur detection
- Exposure analysis
- Composition scoring
- Surface best photos in "Memories"

### Privacy Mode
Extra privacy features:
- Face anonymisation for specific people
- Location stripping for shared albums
- End-to-end encrypted sharing

## Roadmap

### Phase 1: Core
- Immich deployment
- Basic CLIP search
- Mobile app access

### Phase 2: Enhanced
- Face recognition
- Auto-captioning
- Smart albums

### Phase 3: Advanced
- Duplicate detection
- Quality scoring
- Advanced sharing

## FAQs

**Q: Is this really comparable to Google Photos?**
A: Core features yes (search, faces, albums). Google has more polish and some unique features (cinematic photos, Magic Eraser). But for privacy and control, self-hosted wins.

**Q: Can I use this without GPU?**
A: Yes, but processing is slower. Fine for small libraries or if you're patient during initial setup.

**Q: How long does initial processing take?**
A: With GPU: ~1-2 seconds per photo. For 10k photos = ~3-6 hours.
Without GPU: ~5-10 seconds per photo. For 10k photos = ~14-28 hours.

**Q: Can I add more users (family)?**
A: Yes! Immich supports multiple users with separate libraries or shared albums.

**Q: What about video?**
A: Immich handles video great. CLIP works on video frames. More storage needed.

**Q: Can I access from mobile?**
A: Yes! Immich has official iOS and Android apps. Enable Cloudflare Tunnel for secure access.
