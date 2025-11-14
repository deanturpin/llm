# Setup Guide

This guide walks through deploying the personal LLM assistant to a Hetzner VPS with Cloudflare Tunnel.

## Prerequisites

- Hetzner Cloud account
- Domain name (optional, but recommended)
- Cloudflare account (free tier is fine)
- SSH key configured

## Step 1: Provision VPS

### Option A: CPX31 (Budget - £11/month)
```bash
# Via Hetzner Cloud Console or CLI
hcloud server create --name llm-assistant \
    --type cpx31 \
    --image ubuntu-22.04 \
    --ssh-key <your-key>
```

**Specs:** 4 vCPU, 8GB RAM, 160GB SSD

**Suitable for:** Qwen2.5 7B, Llama 3.1 8B, Mistral 7B

### Option B: CCX33 (More power - £44/month)
```bash
hcloud server create --name llm-assistant \
    --type ccx33 \
    --image ubuntu-22.04 \
    --ssh-key <your-key>
```

**Specs:** 8 vCPU, 32GB RAM, 240GB SSD

**Suitable for:** Larger models (Qwen2.5 14B, Llama 3.1 70B quantised)

## Step 2: Initial VPS Configuration

SSH into your VPS:
```bash
ssh root@<vps-ip>
```

Update system and install basics:
```bash
apt update && apt upgrade -y
apt install -y curl git htop
```

Create non-root user (optional but recommended):
```bash
adduser llmadmin
usermod -aG sudo llmadmin
su - llmadmin
```

## Step 3: Deploy Application

From your local machine:

1. **Set up authentication**:
```bash
./scripts/setup-auth.sh
```
Add family member accounts when prompted.

2. **Deploy to VPS**:
```bash
./scripts/deploy.sh llmadmin@<vps-ip>
```

This will:
- Install Docker and Docker Compose
- Copy files to VPS
- Start all services
- Pull the LLM model (takes 10-15 minutes)

## Step 4: Verify Deployment

SSH into VPS and check services:
```bash
ssh llmadmin@<vps-ip>
cd ~/llm/docker
docker-compose ps
```

All services should show "Up" status.

Test the API:
```bash
curl http://localhost/health
```

Should return:
```json
{"status": "healthy", "ollama": "connected", "qdrant": "connected"}
```

## Step 5: Set Up Cloudflare Tunnel

This provides secure access without exposing ports or dealing with firewall configuration.

### On your VPS:

1. **Install cloudflared**:
```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
```

2. **Authenticate**:
```bash
cloudflared tunnel login
```
This opens a browser to authorise.

3. **Create tunnel**:
```bash
cloudflared tunnel create llm-assistant
```
Note the tunnel UUID shown.

4. **Create configuration**:
```bash
cat > ~/.cloudflared/config.yml << EOF
tunnel: <your-tunnel-uuid>
credentials-file: /home/llmadmin/.cloudflared/<your-tunnel-uuid>.json

ingress:
  - hostname: assistant.yourdomain.com  # Change this
    service: http://localhost:80
  - service: http_status:404
EOF
```

5. **Configure DNS**:
```bash
cloudflared tunnel route dns llm-assistant assistant.yourdomain.com
```

6. **Run tunnel as service**:
```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

## Step 6: Test Access

From your local machine:
```bash
curl -u username:password https://assistant.yourdomain.com/health
```

Should return the health check response.

## Step 7: Upload First Document

Test document processing:
```bash
curl -u username:password -X POST \
  -F "file=@/path/to/document.pdf" \
  -F "user_id=username" \
  https://assistant.yourdomain.com/upload
```

## Step 8: Chat Test

```bash
curl -u username:password -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What documents do I have?",
    "user_id": "username",
    "use_context": true
  }' \
  https://assistant.yourdomain.com/chat
```

## Configuration

### Change LLM Model

Edit [config/settings.yml](../config/settings.yml):
```yaml
llm:
  model: "llama3.1:8b"  # or "mistral:7b"
```

Then restart:
```bash
cd ~/llm/docker
docker-compose restart api
docker exec llm-ollama ollama pull llama3.1:8b
```

### Add Family Members

```bash
ssh llmadmin@<vps-ip>
htpasswd -B ~/llm/docker/nginx/.htpasswd newusername
cd ~/llm/docker
docker-compose restart nginx
```

## Maintenance

### View Logs
```bash
cd ~/llm/docker
docker-compose logs -f api
```

### Backup Data
```bash
# Backup Qdrant data
docker run --rm -v llm_qdrant-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/qdrant-backup-$(date +%Y%m%d).tar.gz /data

# Backup uploads
docker run --rm -v llm_uploads:/data -v $(pwd):/backup \
  alpine tar czf /backup/uploads-backup-$(date +%Y%m%d).tar.gz /data
```

### Update Application
```bash
# From local machine
./scripts/deploy.sh llmadmin@<vps-ip>
```

### Monitor Resources
```bash
ssh llmadmin@<vps-ip>
htop
docker stats
```

## Troubleshooting

### Ollama not responding
```bash
docker logs llm-ollama
docker exec llm-ollama ollama list  # Check if model is loaded
```

### Out of memory
- Restart services: `docker-compose restart`
- Consider upgrading VPS or using smaller model
- Check: `docker stats`

### Qdrant connection errors
```bash
docker logs llm-qdrant
curl http://localhost:6333/healthz
```

### Nginx authentication not working
```bash
docker logs llm-nginx
# Check htpasswd file exists
ls -la docker/nginx/.htpasswd
```

## Cost Optimisation

Current monthly costs (CPX31):
- VPS: £11
- Cloudflare: £0 (free tier)
- Domain: £1-2 (optional)

To reduce further:
- Use Hetzner's CX21 (2 vCPU, 4GB RAM) for £5/month
- Run smaller quantised models (Q4_0 instead of Q4_K_M)
- Use Cloudflare for CDN if adding web UI

## Next Steps

1. Implement email integration (see [architecture.md](architecture.md))
2. Add Google Drive sync
3. Build web UI
4. Fine-tune on personal writing style
