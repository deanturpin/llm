# Security Guide for Self-Hosted LLM Systems

This guide covers security best practices for self-hosted LLM deployments, focusing on data protection, access control and operational security.

## Threat Model

When running a self-hosted LLM system, consider these threat vectors:

1. **VPS compromise** - attacker gains access to your server storage
2. **Network interception** - traffic sniffed between client and VPS
3. **Unauthorised access** - family member or external party accessing another user's data
4. **Data exfiltration** - documents or conversations leaked
5. **Supply chain** - compromised Docker images or dependencies

## Defence in Depth

Apply multiple layers of security rather than relying on a single control.

### Layer 1: Network Security

#### Firewall Configuration

Use UFW (Uncomplicated Firewall) to restrict access:

```bash
# Deny all incoming by default
ufw default deny incoming
ufw default allow outgoing

# Allow only necessary ports
ufw allow 22/tcp      # SSH (consider changing default port)
ufw allow 51820/udp   # WireGuard (if using)

# Enable firewall
ufw enable
```

#### VPN Access (Recommended)

**Option 1: Tailscale** (easiest)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Benefits:
- No port forwarding needed
- Automatic encryption
- Works behind NAT
- Free for personal use

**Option 2: WireGuard** (more control)
```bash
sudo apt install wireguard

# Generate keys
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

Create `/etc/wireguard/wg0.conf`:
```ini
[Interface]
PrivateKey = <server_private.key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_public.key>
AllowedIPs = 10.0.0.2/32
```

Enable:
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

### Layer 2: Access Control

#### SSH Hardening

```bash
# Generate SSH key on your local machine (if not already done)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy to VPS
ssh-copy-id user@vps-ip

# On VPS: disable password authentication
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

#### Application Authentication

For HTTP-based APIs, implement multiple authentication layers:

**1. Basic Auth with Nginx**

```nginx
server {
    listen 80;
    server_name _;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Create users:
```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd username
```

**2. API Key Authentication**

Implement in your application:

```python
from fastapi import Security, HTTPException
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

def verify_api_key(api_key: str = Security(api_key_header)):
    # Load valid keys from environment (never hardcode!)
    valid_keys = os.getenv("API_KEYS", "").split(",")
    if api_key not in valid_keys:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key
```

### Layer 3: Data Protection

#### Encryption at Rest

**Volume Encryption with LUKS**

```bash
# Install cryptsetup
sudo apt install cryptsetup

# Create encrypted volume (do this during initial setup!)
sudo cryptsetup luksFormat /dev/sdX
sudo cryptsetup open /dev/sdX encrypted_volume
sudo mkfs.ext4 /dev/mapper/encrypted_volume

# Mount
sudo mkdir -p /encrypted-data
sudo mount /dev/mapper/encrypted_volume /encrypted-data
```

Update docker-compose.yml to use encrypted volume:
```yaml
volumes:
  qdrant-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /encrypted-data/qdrant
```

**Warning**: You'll need to manually unlock after VPS reboot.

#### Document Encryption

Encrypt documents before storing in vector database:

```python
from cryptography.fernet import Fernet
import os
import base64
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2

def get_encryption_key():
    """Derive key from environment variable."""
    password = os.getenv('ENCRYPTION_PASSWORD', '').encode()
    if not password:
        raise ValueError("ENCRYPTION_PASSWORD not set")

    # Use a fixed salt (store this securely)
    salt = os.getenv('ENCRYPTION_SALT', '').encode()

    kdf = PBKDF2(
        algorithm=hashes.SHA256(),
        length=32,
        salt=salt,
        iterations=100000,
    )
    key = base64.urlsafe_b64encode(kdf.derive(password))
    return Fernet(key)

def encrypt_text(text: str) -> str:
    """Encrypt text before storage."""
    cipher = get_encryption_key()
    encrypted = cipher.encrypt(text.encode())
    return base64.urlsafe_b64encode(encrypted).decode()

def decrypt_text(encrypted_text: str) -> str:
    """Decrypt text after retrieval."""
    cipher = get_encryption_key()
    encrypted = base64.urlsafe_b64decode(encrypted_text.encode())
    return cipher.decrypt(encrypted).decode()
```

Usage in upload handler:
```python
@app.post("/upload")
async def upload_document(file: UploadFile, user_id: str):
    # Extract text from document
    text = extract_text(file)

    # Chunk text
    chunks = chunk_text(text)

    # Encrypt and store each chunk
    for chunk in chunks:
        encrypted_chunk = encrypt_text(chunk)
        embedding = embedder.encode(chunk)  # Embed plaintext

        qdrant.upsert(
            collection_name="documents",
            points=[{
                "vector": embedding,
                "payload": {
                    "text": encrypted_chunk,  # Store encrypted
                    "user_id": user_id
                }
            }]
        )

    # Delete original file (don't persist)
    return {"status": "encrypted and stored"}
```

Decrypt during retrieval:
```python
@app.post("/chat")
async def chat(request: ChatRequest):
    # Search for relevant chunks
    results = qdrant.search(...)

    # Decrypt retrieved chunks
    decrypted_chunks = [
        decrypt_text(hit.payload["text"])
        for hit in results
    ]

    # Build context from decrypted text
    context = "\n\n".join(decrypted_chunks)
    # ... continue with LLM call
```

#### Zero-Knowledge Pattern

Even better: don't store original documents at all.

```python
@app.post("/upload")
async def upload_document(file: UploadFile, user_id: str):
    # Process in memory
    text = await extract_text_from_upload(file)
    chunks = chunk_text(text)

    # Store only embeddings and minimal metadata
    for chunk in chunks:
        embedding = embedder.encode(chunk)

        # Store only what's needed for retrieval
        qdrant.upsert(
            collection_name="documents",
            points=[{
                "vector": embedding,
                "payload": {
                    "text": chunk,  # Only the chunk, not full document
                    "user_id": user_id,
                    "uploaded_at": datetime.now().isoformat()
                }
            }]
        )

    # Original file is never written to disk
    # It's processed and discarded

    return {"status": "processed", "chunks": len(chunks)}
```

### Layer 4: Operational Security

#### Environment Variables

Never commit secrets to git:

```bash
# On VPS, create .env file
cat > .env << EOF
ENCRYPTION_PASSWORD=your-secure-password-here
ENCRYPTION_SALT=your-random-salt-here
API_KEYS=key1,key2,key3
QDRANT_API_KEY=your-qdrant-key
EOF

chmod 600 .env
```

Load in docker-compose.yml:
```yaml
services:
  api:
    env_file: .env
```

#### Regular Updates

```bash
# Create update script
cat > /home/llmadmin/update.sh << 'EOF'
#!/bin/bash
set -e

echo "Updating system packages..."
sudo apt update && sudo apt upgrade -y

echo "Updating Docker images..."
cd ~/llm-pa
docker-compose pull
docker-compose up -d

echo "Pruning old images..."
docker image prune -f

echo "Update complete!"
EOF

chmod +x /home/llmadmin/update.sh

# Add to cron (weekly updates)
(crontab -l 2>/dev/null; echo "0 3 * * 0 /home/llmadmin/update.sh >> /var/log/update.log 2>&1") | crontab -
```

#### Fail2Ban

Protect against brute force attacks:

```bash
sudo apt install fail2ban

# Create SSH jail
sudo cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
EOF

sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

#### Audit Logging

Log all API access:

```python
import logging
from datetime import datetime

logging.basicConfig(
    filename='/var/log/pa-api.log',
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = datetime.now()

    # Log request
    logging.info(f"Request: {request.method} {request.url.path} from {request.client.host}")

    response = await call_next(request)

    # Log response
    duration = (datetime.now() - start_time).total_seconds()
    logging.info(f"Response: {response.status_code} in {duration:.2f}s")

    return response
```

#### Backup Strategy

Encrypted backups to separate location:

```bash
#!/bin/bash
# /home/llmadmin/backup.sh

BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/qdrant-$DATE.tar.gz"

# Backup Qdrant data
docker run --rm \
  -v qdrant-data:/data \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/qdrant-$DATE.tar.gz /data

# Encrypt backup
gpg --encrypt --recipient your-email@example.com $BACKUP_FILE

# Delete unencrypted backup
rm $BACKUP_FILE

# Upload to remote storage (example: rsync to another server)
rsync -avz $BACKUP_FILE.gpg backup-server:/backups/

# Keep only last 7 days locally
find $BACKUP_DIR -name "qdrant-*.gpg" -mtime +7 -delete

echo "Backup completed: $BACKUP_FILE.gpg"
```

## User Isolation

Ensure users can only access their own data:

```python
@app.post("/chat")
async def chat(request: ChatRequest, user_id: str = Depends(get_current_user)):
    # Always filter by authenticated user_id
    results = qdrant.search(
        collection_name="documents",
        query_vector=query_vector,
        query_filter={
            "must": [
                {"key": "user_id", "match": {"value": user_id}}
            ]
        }
    )
```

Never trust client-provided user_id - always derive from authentication.

## Security Checklist

- [ ] SSH key-based authentication only
- [ ] Root login disabled
- [ ] Firewall configured (UFW)
- [ ] VPN access (Tailscale or WireGuard)
- [ ] HTTPS with valid certificate (if exposing to internet)
- [ ] Environment variables for secrets
- [ ] User isolation in database queries
- [ ] Regular system updates automated
- [ ] Fail2Ban configured
- [ ] Audit logging enabled
- [ ] Encrypted backups configured
- [ ] Document encryption (optional but recommended)
- [ ] Volume encryption (for sensitive data)

## Monitoring

Set up basic monitoring:

```bash
# Install monitoring tools
sudo apt install htop iotop nethogs

# Check disk usage
df -h

# Check Docker resource usage
docker stats

# Check logs
docker-compose logs -f --tail=100
```

## Incident Response

If you suspect compromise:

1. **Immediately**: Shut down services
   ```bash
   docker-compose down
   ```

2. **Investigate**: Check logs
   ```bash
   sudo grep -i "failed\|error\|unauthorized" /var/log/auth.log
   journalctl -xe
   ```

3. **Rotate credentials**: Change all passwords and API keys

4. **Restore from backup**: If data integrity is questioned

5. **Update and patch**: Ensure all systems are current

6. **Document**: Record what happened for future prevention

## Privacy by Design

**Principle**: Collect and store only what you need.

- Don't log conversation content (only metadata)
- Delete documents after embedding if not needed for re-indexing
- Set retention policies (auto-delete old data)
- Implement user data export functionality
- Provide user data deletion on request

## Further Reading

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Qdrant Security Features](https://qdrant.tech/documentation/security/)

## Legal Considerations

If running in the UK:
- Comply with UK GDPR for family data
- Implement data subject rights (access, deletion)
- Consider Data Protection Impact Assessment (DPIA) for sensitive data
- Maintain records of processing activities

**This guide is educational. Adapt to your specific threat model and risk tolerance.**
