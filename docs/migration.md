# Migrating Between VPS Providers

Your personal assistant is **fully portable**. You can move between providers or to a home server with minimal downtime.

## What Makes It Portable?

### ✅ Everything is containerised
- Docker containers work identically on any Linux host
- Same `docker-compose.yml` works everywhere

### ✅ Data is in volumes
- All important data in Docker volumes
- Easy to backup and restore
- Independent of the host system

### ✅ No vendor lock-in
- No provider-specific features used
- Standard Ubuntu/Debian setup
- Open source stack throughout

## Migration Scenarios

### 1. Provider to Provider (e.g., Fasthosts → Hetzner)
**Reason**: Cost optimisation, better performance, different location

**Downtime**: 30-60 minutes

### 2. VPS to Home Server
**Reason**: Maximum privacy, long-term cost savings

**Downtime**: 1-2 hours (first time setup)

### 3. Home Server to VPS
**Reason**: Better uptime, internet issues, moving house

**Downtime**: 30-60 minutes

### 4. Provider Upgrade (Same Provider)
**Reason**: Need more RAM/CPU

**Downtime**: 15-30 minutes (often zero with proper planning)

## Migration Process

### Step 1: Backup Current System

On your **old VPS**:

```bash
# Stop services (optional, for consistency)
cd ~/llm/docker
docker-compose stop

# Backup Docker volumes
docker run --rm \
  -v llm_ollama-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/ollama-backup.tar.gz /data

docker run --rm \
  -v llm_qdrant-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/qdrant-backup.tar.gz /data

docker run --rm \
  -v llm_uploads:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/uploads-backup.tar.gz /data

# Backup configuration
tar czf config-backup.tar.gz ~/llm

# Download backups to your local machine
scp old-vps:~/llm/docker/*-backup.tar.gz ./
scp old-vps:config-backup.tar.gz ./
```

**What's backed up:**
- ✅ Ollama models (can be large: 4-8GB per model)
- ✅ Qdrant vector database (your document embeddings)
- ✅ Uploaded documents
- ✅ Configuration files
- ✅ User credentials

### Step 2: Provision New VPS

**On new VPS:**

```bash
# Same as initial setup
apt update && apt upgrade -y
apt install -y curl git htop ufw

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Create user
adduser llmadmin
usermod -aG sudo,docker llmadmin
```

### Step 3: Transfer Configuration

**From your local machine:**

```bash
# Upload configuration
scp config-backup.tar.gz new-vps:~/
ssh new-vps "cd ~ && tar xzf config-backup.tar.gz"
```

### Step 4: Restore Data

**On new VPS:**

```bash
# Upload backups
# (from local machine)
scp *-backup.tar.gz new-vps:~/llm/docker/

# On new VPS - start services first to create volumes
cd ~/llm/docker
docker-compose up -d

# Stop them immediately
docker-compose stop

# Restore volumes
docker run --rm \
  -v llm_ollama-data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/ollama-backup.tar.gz --strip 1"

docker run --rm \
  -v llm_qdrant-data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/qdrant-backup.tar.gz --strip 1"

docker run --rm \
  -v llm_uploads:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/uploads-backup.tar.gz --strip 1"

# Restart services with restored data
docker-compose up -d
```

### Step 5: Verify Migration

```bash
# Check all services are running
docker-compose ps

# Test Ollama
docker exec llm-ollama ollama list

# Test Qdrant
curl http://localhost:6333/healthz

# Test API
curl http://localhost:8000/health

# Test a query (with your credentials)
curl -u username:password http://localhost:8000/health
```

### Step 6: Update Access Method

#### If using Cloudflare Tunnel:

```bash
# On new VPS
cloudflared tunnel login
cloudflared tunnel create pa-new

# Update DNS to point to new tunnel
cloudflared tunnel route dns pa-new pa.yourdomain.com

# Run as service
cloudflared service install
systemctl start cloudflared
```

#### If using WireGuard:

```bash
# Update your WireGuard config with new VPS IP
# On your laptop: edit /etc/wireguard/wg0.conf
# Change Endpoint to new-vps-ip:51820

# Restart WireGuard
wg-quick down wg0
wg-quick up wg0
```

#### If using Tailscale:

```bash
# On new VPS
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Automatically shows up in your network
# Access by hostname (vps-name)
```

### Step 7: Decommission Old VPS

**After confirming new VPS works:**

```bash
# On old VPS - cleanup
docker-compose down -v  # Remove volumes
rm -rf ~/llm
rm *-backup.tar.gz

# Then cancel/delete old VPS through provider
```

## Migration Time Breakdown

### Total Migration Time

| Step | Duration | Can Parallelize? |
|------|----------|------------------|
| 1. Backup old VPS | 15-20 min | - |
| 2. Provision new VPS | 5-10 min | ✅ (while backing up) |
| 3. Transfer config | 2-5 min | - |
| 4. Restore data | 10-15 min | - |
| 5. Verify | 5 min | - |
| 6. Update access | 5-10 min | - |
| **Total** | **40-60 min** | **(30 min with parallelization)** |

### Downtime

**With planning**: ~5-15 minutes
- Time between shutting down old and new being accessible
- Can reduce to near-zero with proper planning

**Without planning**: 30-60 minutes
- If discovering issues during migration

## Zero-Downtime Migration (Advanced)

### Strategy

Run both VPSs in parallel briefly:

```
Old VPS ─────────┐
                 ├─→ Switch over ─→ New VPS
New VPS ─────────┘
```

### Process

1. **Backup old VPS** (while still running)
2. **Provision and setup new VPS** with backup
3. **Pause writes on old VPS** (put in read-only mode)
4. **Sync final changes** (incremental backup)
5. **Switch DNS/access** to new VPS
6. **Verify** new VPS working
7. **Decommission** old VPS

**Downtime**: ~30 seconds (DNS propagation)

### Implementation

```bash
# On old VPS - read-only mode (custom implementation)
# Stop accepting uploads, finish existing requests
docker-compose exec api /app/scripts/enable-readonly-mode.sh

# Quick incremental sync
rsync -avz old-vps:~/llm/docker/volumes/ new-vps:~/llm/docker/volumes/

# Switch DNS immediately
# Update Cloudflare Tunnel or WireGuard endpoint

# Old VPS can stay running for a day as backup
```

## Migration Cost Considerations

### Overlapping Billing

Most providers bill monthly, so you might pay for both temporarily:

**Example: Fasthosts to Hetzner**
- Keep Fasthosts for 1 week overlap (£5-7 wasted)
- Ensure smooth migration
- Decommission after confirming stability

### Minimising Cost

1. **Same provider upgrade**: Often can upgrade in-place
2. **Time it well**: Migrate near billing cycle start
3. **Snapshot/restore**: Some providers offer instant snapshots

## Provider-Specific Notes

### Fasthosts → Hetzner

**Pros:**
- ✅ Save £9-14/month (Fasthosts £20-25 → Hetzner £11)
- ✅ Better performance (NVMe storage)
- ✅ Good tooling (hcloud CLI)

**Cons:**
- ⚠️ UK → Germany (data residency)
- ⚠️ Different support culture

### Hetzner → Fasthosts

**Pros:**
- ✅ UK data residency
- ✅ UK-based support

**Cons:**
- ❌ More expensive (£11 → £20-25)
- ⚠️ Potentially slower storage

### Any VPS → Home Server

**Pros:**
- ✅ Maximum privacy and control
- ✅ Long-term cost savings
- ✅ No monthly fees

**Cons:**
- ⚠️ Dependent on home internet
- ⚠️ You handle hardware failures
- ⚠️ Initial hardware cost (£500-2000)

### Home Server → VPS

**Pros:**
- ✅ Better uptime
- ✅ Professional infrastructure
- ✅ No hardware maintenance

**Cons:**
- ❌ Ongoing monthly cost
- ⚠️ Less privacy than home (but still private)

## Data Portability Checklist

Before migrating, ensure you have:

- [ ] All Docker volumes backed up
- [ ] Configuration files backed up
- [ ] User credentials backed up (.htpasswd file)
- [ ] Ollama models list (`ollama list`)
- [ ] Qdrant collections list
- [ ] Any custom scripts or modifications
- [ ] Backup stored locally (not just on old VPS)
- [ ] Access method documented (WireGuard keys, etc.)

## Testing Migration (Dry Run)

### Practice on a Second VPS

Before migrating production:

1. Spin up a test VPS (Hetzner has €20 credit for new accounts)
2. Run through migration process
3. Time each step
4. Identify issues
5. Delete test VPS
6. Run real migration confidently

**Cost**: €0-5 (one day of testing)
**Benefit**: Confidence and faster real migration

## Backup Schedule (Prevent Data Loss)

Don't wait until migration to have backups:

```bash
# Automated daily backups
# Add to crontab: crontab -e

# Daily backup at 2 AM
0 2 * * * /home/llmadmin/llm/scripts/daily-backup.sh

# Weekly backup to off-site
0 3 * * 0 /home/llmadmin/llm/scripts/weekly-offsite-backup.sh
```

### Backup Script Example

```bash
#!/bin/bash
# daily-backup.sh

BACKUP_DIR="/home/llmadmin/backups"
DATE=$(date +%Y%m%d)

# Backup volumes
docker run --rm \
  -v llm_qdrant-data:/data \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/qdrant-$DATE.tar.gz /data

# Keep last 7 days
find $BACKUP_DIR -name "qdrant-*.tar.gz" -mtime +7 -delete

# Encrypt and upload to Wasabi/Backblaze (optional)
gpg --encrypt --recipient you@example.com \
  $BACKUP_DIR/qdrant-$DATE.tar.gz

rclone copy $BACKUP_DIR/qdrant-$DATE.tar.gz.gpg wasabi:backups/
```

## When to Migrate

### Good Reasons

✅ **Cost savings** (e.g., Fasthosts → Hetzner)
✅ **Performance issues** (need more RAM/CPU)
✅ **Data residency** requirements change
✅ **Moving to home server** (long-term plan)
✅ **Provider reliability** concerns

### Bad Reasons

❌ **Boredom** (unnecessary risk)
❌ **Minimal savings** (< £5/month not worth effort)
❌ **Impulse** (plan migrations, don't rush)

## FAQ

**Q: Can I migrate while the PA is in use?**
A: Yes, but ideally do it during low-usage time (night/weekend). Brief downtime is unavoidable unless doing zero-downtime approach.

**Q: Will I lose my chat history?**
A: No, if you backup and restore the Qdrant volume. All conversation context is stored there.

**Q: Do I need to re-download LLM models?**
A: No, if you backup the Ollama volume. Models are several GB, so backup/restore is faster than re-downloading.

**Q: What if the migration fails?**
A: Your old VPS is still running with everything intact. Just keep using it and troubleshoot the new one.

**Q: Can I run both simultaneously?**
A: Yes! Useful for testing or zero-downtime migration. Just ensure you know which one is "active" to avoid confusion.

**Q: How often should I migrate?**
A: Rarely. Only when there's a good reason (cost, performance, requirements change). Your PA is stable infrastructure, not a hobby project to constantly tinker with.

**Q: Is snapshot/restore faster?**
A: Yes, if both providers support it. Hetzner has snapshots. But manual backup/restore is more portable (works between any providers).

## Summary

**Migration is easy because:**
- ✅ Everything is containerised
- ✅ Data is in clearly defined volumes
- ✅ No vendor-specific features
- ✅ Standard Docker Compose setup
- ✅ Open source stack throughout

**Your data is never locked to a provider.**

Pick the VPS that makes sense now, migrate later if needs change. The architecture is designed for portability from day one.
