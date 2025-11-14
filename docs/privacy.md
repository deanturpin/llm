# Privacy and Data Security

## How Your Data Stays Private

### When You Self-Host with Ollama

**Your data NEVER leaves your infrastructure.** Here's why:

1. **Local Model Inference**
   - The LLM runs entirely on your VPS
   - No API calls to external services
   - No telemetry or usage tracking
   - Model weights downloaded once, then offline

2. **Ollama is Offline**
   - Ollama only phones home to check for updates (can be disabled)
   - Model inference is 100% local
   - Your prompts and responses never transmitted
   - Open source - you can audit the code

3. **Vector Database (Qdrant)**
   - Stores document embeddings locally
   - No external connections
   - Can be encrypted at rest
   - Isolated to your VPS

### Comparison: API-Based LLMs

| Service | Data Privacy | How It Works |
|---------|--------------|--------------|
| **ChatGPT/Claude API** | ❌ Data sent to provider | Your prompt → OpenAI/Anthropic servers → Response |
| **Ollama (self-hosted)** | ✅ Data stays local | Your prompt → Your VPS → Response |

### What Gets Sent Where?

#### ✅ Stays on Your VPS:
- Your uploaded documents
- Your chat queries
- All LLM prompts and responses
- Document embeddings
- Conversation history
- User data

#### ⚠️ May Leave VPS (if enabled):
- Cloudflare Tunnel: Encrypted tunnel metadata (not your data)
- OS updates: Standard package manager traffic
- Docker pulls: Image downloads (one-time)
- Monitoring (if you set it up): Metrics only, not data

#### ❌ NEVER Leaves:
- Your actual document contents
- LLM training data (models downloaded once)
- Your queries and responses
- Personal information

## Email Access Considerations

### Should You Give PA Read Access to All Emails?

**Start conservative, expand carefully.**

### Option 1: Manual Upload (Most Private)
```
You → Export specific emails as .eml files → Upload to PA
```

**Pros:**
- Maximum control
- Review before sharing
- No automated access

**Cons:**
- Manual process
- Less convenient

### Option 2: IMAP Read-Only (Automated)
```
PA → IMAP connection → Read emails → Extract and index
```

**Pros:**
- Automatic indexing
- Always up-to-date
- More useful assistant

**Cons:**
- PA has access to all email
- Credentials stored on VPS

### Option 3: Filtered IMAP (Balanced)
```
PA → IMAP connection → Read specific labels/folders only
```

**Pros:**
- Automatic for relevant emails
- Controlled scope
- Balance of convenience and privacy

**Cons:**
- Requires email filtering setup

### Recommendation: Option 3

Set up email filters in your client:
- Label "PA" on emails you want indexed
- PA only reads emails with this label
- You control what it sees

### Implementation
```python
# In your PA config
email:
  enabled: true
  imap_host: imap.gmail.com
  username: you@gmail.com
  password: app-specific-password
  folders:
    - "PA"  # Only index this label
  # Never index these folders
  exclude:
    - "Banking"
    - "Private"
    - "Work Confidential"
```

### Email Security Best Practices

1. **Use App-Specific Passwords**
   - Never store your main email password
   - Generate app password in Gmail/Outlook
   - Can revoke anytime

2. **Encrypt Credentials**
   ```bash
   # Store encrypted on VPS
   echo "password" | gpg --encrypt > email.gpg
   ```

3. **Read-Only Access**
   - IMAP read-only (can't send, delete, or modify)
   - PA never writes back to email

4. **Regular Audits**
   - Review what's been indexed
   - Check access logs
   - Rotate credentials quarterly

## Infrastructure Independence

### FastHosts VPS and Cloud Independence

Good news! **Your setup is completely independent of Google/Amazon.**

### Your Stack:
```
FastHosts VPS (your choice)
    ↓
Ubuntu/Debian (open source)
    ↓
Docker (open source)
    ↓
Ollama (open source)
    ↓
Qwen/Llama models (open source)
    ↓
Your data (stays local)
```

**Zero Google/Amazon dependencies!**

### What About Cloudflare?

**Cloudflare Tunnel** is optional but recommended:
- Free tier (no cost)
- Only handles encrypted routing
- Sees encrypted traffic (can't read your data)
- Alternative: Tailscale (also good, different approach)

If you want **zero external services**:
```
FastHosts VPS
    ↓
Direct SSH access (for you)
    ↓
WireGuard VPN (open source)
    ↓
Access from your devices
```

### FastHosts Specifics

**What FastHosts sees:**
- VM metrics (CPU, RAM, disk)
- Network traffic volumes
- Nothing about content

**What they DON'T see:**
- Your encrypted disk contents
- Your prompts and responses
- Your document data

### Making It Even More Private

#### 1. Full Disk Encryption
```bash
# Set up LUKS encryption on VPS
cryptsetup luksFormat /dev/sdb
cryptsetup luksOpen /dev/sdb encrypted-data
# Store Docker volumes here
```

#### 2. Encrypted Backups
```bash
# Backup with encryption
tar czf - /var/lib/docker/volumes | gpg --encrypt > backup.tar.gz.gpg
# Only you can decrypt with your GPG key
```

#### 3. Wireguard Instead of Cloudflare
```bash
# Install WireGuard on VPS
apt install wireguard

# Configure peer (your laptop)
wg-quick up wg0

# Now access VPS through encrypted tunnel
# No third-party service involved
```

## Model Trust

### "How Do I Know the Model Isn't Stealing Data?"

**Open source models are auditable:**

1. **Ollama is open source**: https://github.com/ollama/ollama
   - You can read the code
   - Community audits it
   - No telemetry by default

2. **Model weights are frozen**:
   - Models downloaded once
   - Inference is deterministic
   - Can't "phone home" (no network calls)
   - Can run airgapped after initial download

3. **You can verify**:
   ```bash
   # Run Ollama with network disabled
   docker run --network none ollama/ollama
   # Still works! Proof it's offline
   ```

### What About Model Updates?

**You control when to update:**
```bash
# Check current model
docker exec llm-ollama ollama list

# Update only when YOU choose
docker exec llm-ollama ollama pull qwen2.5:7b

# Pin specific version
# qwen2.5:7b-instruct-q4_K_M (specific quantisation)
```

## Maximum Privacy Configuration

If you're paranoid (reasonably!), here's the setup:

### 1. Airgapped Initial Setup
```bash
# On internet-connected machine
docker pull ollama/ollama
docker pull qdrant/qdrant
ollama pull qwen2.5:7b

# Export images
docker save ollama/ollama > ollama.tar
docker save qdrant/qdrant > qdrant.tar

# Transfer to airgapped VPS
scp *.tar vps:/tmp/

# On VPS (now offline)
docker load < /tmp/ollama.tar
docker load < /tmp/qdrant.tar
```

### 2. Disable Ollama Update Checks
```yaml
# docker-compose.yml
environment:
  - OLLAMA_SKIP_UPDATES=true
```

### 3. Firewall Everything
```bash
# Only allow SSH and your VPN
ufw default deny incoming
ufw default deny outgoing
ufw allow ssh
ufw allow from <your-wireguard-subnet>
ufw enable
```

### 4. Audit Everything
```bash
# Log all network connections
apt install auditd
auditctl -a exit,always -F arch=b64 -S connect

# Review logs
ausearch -sc connect
```

## Trust Model Summary

**You must trust:**
- Your VPS provider (FastHosts) - but they only see encrypted data
- Open source software (Ollama, Qdrant) - auditable
- Your own security practices

**You do NOT need to trust:**
- Google, Amazon, Microsoft
- OpenAI, Anthropic, Google AI
- Any external API services
- Model providers (after download)

## Privacy Checklist

- [ ] VPS with reputable provider (FastHosts ✓)
- [ ] Full disk encryption enabled
- [ ] Ollama update checks disabled (optional)
- [ ] Email access limited to specific folders
- [ ] App-specific passwords (not main password)
- [ ] WireGuard or Cloudflare Tunnel for access
- [ ] Firewall configured
- [ ] Regular backups (encrypted)
- [ ] Monitoring (no external services)
- [ ] Audit logs reviewed monthly

## Conclusion

**Self-hosting with Ollama is genuinely private.** Your data never touches Google, Amazon, or any AI company servers. The only entity with physical access is your VPS provider, and they see encrypted data.

This is **fundamentally different** from using ChatGPT, Claude, or Gemini APIs where your data is sent to the provider.

**FastHosts VPS + Ollama = Maximum privacy** (within practical constraints).
