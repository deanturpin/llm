# Infrastructure Guide

## VPS Provider Options

### Hetzner (Recommended in Main Guide)
**Why mentioned**: Excellent price/performance, popular in EU

**Pros:**
- Great value
- German data centres
- Good reputation
- Easy API/CLI

**Cons:**
- Uses Hetzner infrastructure (obviously)
- EU-based (may matter for some)

### FastHosts (Your Choice)
**Why it works**: UK-based, no Google/Amazon infrastructure

**Pros:**
- UK data centres
- Independent infrastructure
- Good UK support
- No big tech dependencies

**Cons:**
- Typically more expensive than Hetzner
- Smaller ecosystem

### Other Independent Providers

#### OVH
- French company
- Own data centres
- Good VPS options
- Independent of AWS/GCP/Azure

#### Scaleway
- French company
- Owned by Iliad/Free
- Competitive pricing
- Independent infrastructure

#### Linode (Akamai)
- Now owned by Akamai
- Independent infrastructure
- Global presence
- Good documentation

#### DigitalOcean
⚠️ **Uses Amazon and Google cloud in some regions**
- Check specific data centre
- Not fully independent

### Fully Independent Stack

If you want **zero** reliance on big tech:

```
Your Hardware (home server)
    ↓
Linux (Debian/Ubuntu)
    ↓
Docker
    ↓
Ollama + Qdrant
    ↓
Your data
```

**Pros:**
- Complete control
- No monthly costs (after hardware)
- Lowest latency
- True data sovereignty

**Cons:**
- Upfront hardware cost (£500-2000)
- Power consumption (~£5-10/month)
- Internet uplink dependency
- You handle hardware failures

## Infrastructure Comparison

### For Personal Assistant

| Provider | Spec | Cost/Month | Independence | Notes |
|----------|------|------------|--------------|-------|
| **Hetzner CPX31** | 4 vCPU, 8GB | £11 | ⭐⭐⭐⭐ | Best value |
| **FastHosts VPS** | 4 vCPU, 8GB | £15-25 | ⭐⭐⭐⭐⭐ | UK-based, independent |
| **OVH VPS Elite** | 4 vCPU, 8GB | £18 | ⭐⭐⭐⭐ | French, independent |
| **Home Server** | Mini PC | £500+ (one-time) | ⭐⭐⭐⭐⭐ | Full control |

### For Photo Management (needs more power)

| Provider | Spec | Cost/Month | Independence | Notes |
|----------|------|------------|--------------|-------|
| **Hetzner CCX33** | 8 vCPU, 32GB | £44 | ⭐⭐⭐⭐ | Good for CPU inference |
| **FastHosts VPS** | 8 vCPU, 32GB | £60-80 | ⭐⭐⭐⭐⭐ | UK independent |
| **Home Server + GPU** | RTX 3060 | £800+ (one-time) | ⭐⭐⭐⭐⭐ | Best performance |

## FastHosts-Specific Setup

### Provisioning a FastHosts VPS

1. **Choose OS**: Ubuntu 22.04 LTS (most tested)
2. **Size**: Start with 8GB RAM minimum
3. **Location**: UK data centre
4. **Backups**: Enable automated backups

### Initial Configuration

```bash
# SSH into your new VPS
ssh root@your-vps-ip

# Update system
apt update && apt upgrade -y

# Install essentials
apt install -y curl git htop ufw fail2ban

# Set up firewall
ufw default deny incoming
ufw default deny outgoing
ufw allow ssh
ufw allow http
ufw allow https
ufw enable

# Create non-root user
adduser llmadmin
usermod -aG sudo llmadmin
```

### Docker Installation

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Add user to docker group
usermod -aG docker llmadmin

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Verify
docker --version
docker-compose --version
```

## Network Access Options

### Option 1: Cloudflare Tunnel (Easiest)

**Pros:**
- Free
- No port forwarding
- DDoS protection
- Automatic TLS

**Cons:**
- Depends on Cloudflare (US company)
- They see encrypted traffic metadata

**Setup:**
```bash
# Install cloudflared
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared.deb

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create my-pa

# Configure DNS
cloudflared tunnel route dns my-pa pa.yourdomain.com

# Run as service
cloudflared service install
```

### Option 2: WireGuard (Most Private)

**Pros:**
- Open source
- Encrypted peer-to-peer
- No third party
- Fast

**Cons:**
- Manual setup on each device
- No web UI (just VPN)

**Setup:**
```bash
# On VPS
apt install wireguard

# Generate keys
wg genkey | tee server_private.key | wg pubkey > server_public.key

# Configure /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <server_private.key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_public.key>
AllowedIPs = 10.0.0.2/32

# Enable
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0

# Open firewall
ufw allow 51820/udp
```

**On Your Laptop:**
```bash
# macOS
brew install wireguard-tools

# Configure client
# Then access VPS services via 10.0.0.1
```

### Option 3: Tailscale (Easiest VPN)

**Pros:**
- Dead simple setup
- Works everywhere
- Free for personal use
- WireGuard under the hood

**Cons:**
- Depends on Tailscale (coordination server)
- They can't see data, but see metadata

**Setup:**
```bash
# On VPS
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# On your laptop
# Install Tailscale app
# Login with same account

# Access VPS by name
curl http://vps-name:8000
```

### Comparison

| Method | Privacy | Ease of Use | Cost | Family Access |
|--------|---------|-------------|------|---------------|
| **Cloudflare Tunnel** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Free | Easy (web URL) |
| **WireGuard** | ⭐⭐⭐⭐⭐ | ⭐⭐ | Free | Medium (install app) |
| **Tailscale** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Free | Easy (install app) |
| **Direct SSH** | ⭐⭐⭐⭐⭐ | ⭐ | Free | Hard (terminal only) |

**Recommendation for You (FastHosts + Privacy Focus):**
- **Primary**: WireGuard (maximum privacy)
- **Fallback**: Tailscale (easier for family)
- **Avoid**: Cloudflare Tunnel (since you're avoiding big infrastructure)

## Storage Options

### Local VPS Disk
- Included with VPS
- Fast
- Limited size
- Good for: OS, Docker images, active data

### FastHosts Storage Volumes
- Block storage attached to VPS
- Fast (SSD)
- Scalable
- Good for: Photo libraries, large document sets

### External S3-Compatible Storage

**NOT Recommended** if avoiding big tech:
- ❌ AWS S3 (Amazon)
- ❌ Google Cloud Storage (Google)
- ❌ Azure Blob Storage (Microsoft)

**Independent alternatives:**
- ✅ Wasabi (independent, S3-compatible)
- ✅ Backblaze B2 (independent)
- ✅ Your own MinIO instance

### Backup Strategy

```bash
# Local encrypted backup
tar czf - /var/lib/docker/volumes | \
  gpg --encrypt --recipient you@example.com > \
  backup-$(date +%Y%m%d).tar.gz.gpg

# Transfer off-site
scp backup-*.gpg your-home-nas:/backups/
# Or to independent cloud storage
rclone copy backup-*.gpg wasabi:backups/
```

## Monitoring (Without External Services)

### Self-Hosted Monitoring Stack

```yaml
# docker-compose-monitoring.yml
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=your-secure-password

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
```

**Access via WireGuard:** `http://10.0.0.1:3000`

### Simple Monitoring Script

```bash
#!/bin/bash
# monitor.sh - Simple resource monitoring

while true; do
    echo "=== $(date) ===" >> /var/log/vps-monitor.log

    # CPU and memory
    top -bn1 | grep "Cpu(s)" >> /var/log/vps-monitor.log
    free -h >> /var/log/vps-monitor.log

    # Docker stats
    docker stats --no-stream >> /var/log/vps-monitor.log

    # Disk
    df -h >> /var/log/vps-monitor.log

    echo "" >> /var/log/vps-monitor.log

    sleep 300  # Every 5 minutes
done
```

Run as systemd service:
```ini
# /etc/systemd/system/vps-monitor.service
[Unit]
Description=VPS Monitoring

[Service]
ExecStart=/usr/local/bin/monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

## Home Server Option

### Hardware Recommendations

#### Budget Option (~£500)
- **Intel NUC or similar mini PC**
- 32GB RAM (upgradeable)
- 1TB NVMe SSD
- Power: ~15W idle, ~30W load

#### Performance Option (~£1200)
- **Custom build with GPU**
- AMD Ryzen 7/9 or Intel i7/i9
- 64GB RAM
- RTX 3060 or better
- 2TB NVMe SSD
- Power: ~50W idle, ~250W load

#### Premium Option (~£2500+)
- **Workstation-class**
- Threadripper or Xeon
- 128GB+ RAM
- RTX 4090 or professional GPU
- Multi-TB NVMe RAID
- Proper server chassis

### Power Cost Calculation

```
Budget server: 30W average
30W × 24h × 30 days = 21.6 kWh/month
At £0.30/kWh (UK avg) = £6.48/month

Performance server: 150W average
150W × 24h × 30 days = 108 kWh/month
At £0.30/kWh = £32.40/month
```

### Break-Even Analysis

**Budget Home Server (£500 + £6.50/month power) vs FastHosts VPS (£25/month)**

```
Month 1: £500 + £6.50 vs £25 (hardware amortised over time)
Break-even: ~27 months

After 3 years:
Home: £500 + (£6.50 × 36) = £734
VPS: £25 × 36 = £900

You save: £166 (plus you own the hardware)
```

### Remote Access from Home Server

**Option 1: Dynamic DNS + VPN**
```bash
# Use ddclient for dynamic IP
apt install ddclient

# Configure with No-IP, DynDNS, etc.
# Then WireGuard for secure access
```

**Option 2: Tailscale**
- Easiest for home server
- Handles NAT traversal
- Works behind any router

**Option 3: Cloudflare Tunnel**
- Works even with restrictive ISP
- But depends on Cloudflare

## Complete Independent Stack

**Maximum privacy and independence:**

```
Hardware: Your own server or independent VPS (FastHosts, OVH)
    ↓
OS: Linux (open source)
    ↓
Container: Docker (open source)
    ↓
LLM Runtime: Ollama (open source)
    ↓
Model: Qwen/Llama (open source, open weights)
    ↓
Vector DB: Qdrant (open source)
    ↓
Access: WireGuard (open source) or Tailscale
    ↓
Monitoring: Prometheus + Grafana (open source, self-hosted)
    ↓
Backups: GPG encrypted, stored locally or Wasabi/Backblaze
```

**Zero dependence on:**
- ❌ Google
- ❌ Amazon
- ❌ Microsoft
- ❌ OpenAI
- ❌ Anthropic

**Auditable:**
- ✅ Every component is open source
- ✅ You can read the code
- ✅ Community audits
- ✅ No telemetry

## UK-Specific Considerations

### Data Residency
- FastHosts: UK data centres ✅
- Hetzner: German/Finnish data centres
- OVH: French data centres

### GDPR Compliance
- All EU/UK providers are GDPR-compliant
- Self-hosting gives you maximum control
- You are the data controller

### ISP Considerations (for home server)
- Check for CGNAT (some UK ISPs)
- May need static IP (£5/month extra)
- Check upload speed (for family access)

## Recommendations Summary

### For Maximum Privacy (Your Use Case)
1. **Infrastructure**: FastHosts VPS (UK, independent)
2. **Access**: WireGuard (no third party)
3. **Monitoring**: Self-hosted Prometheus
4. **Backups**: GPG encrypted to Wasabi or local NAS
5. **No** Google/Amazon/Microsoft anywhere

### For Best Value
1. **Infrastructure**: Hetzner CPX31
2. **Access**: Cloudflare Tunnel (easiest)
3. **Monitoring**: Simple scripts
4. **Backups**: Hetzner Storage Box

### For Maximum Control
1. **Infrastructure**: Home server (one-time cost)
2. **Access**: Tailscale (easiest) or WireGuard (most private)
3. **Monitoring**: Full Prometheus + Grafana stack
4. **Backups**: Local NAS + off-site encrypted

Choose based on your priorities: privacy, cost, or control!
