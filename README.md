# RAGFlow-HTM

AI-powered service manual knowledge base for Healthcare Technology Management (HTM) engineers.

Upload device service manuals (PDF, Word, Excel, scanned images) and ask natural language questions like:

> "Philips MX800 error code 42 是什么意思？"
> "What is the PM schedule for GE Carescape R860?"
> "Siemens Artis zee troubleshooting steps for error E204"

Built on [RAGFlow](https://github.com/infiniflow/ragflow) — an open-source RAG engine with deep document understanding.

## System Requirements

A Linux VPS with:

| Resource | Minimum |
|----------|---------|
| CPU | 4 cores |
| RAM | 16 GB |
| Disk | 50 GB |
| Docker | 24.0.0+ |
| Docker Compose | v2.26.1+ |

## Deployment Overview

```
User (browser)
     │
     │  https://htm.localhub.nz
     ▼
Cloudflare (DNS + SSL)
     │
     │  Cloudflare Tunnel
     ▼
VPS (no open ports needed)
     │
     │  localhost:80
     ▼
RAGFlow (Docker Compose)
```

## Step-by-Step Deployment

### 1. Get a VPS

Any Linux VPS with 4 cores / 16GB RAM / 50GB disk. Example providers:

| Provider | Config | ~Cost/mo |
|----------|--------|----------|
| Hetzner | CPX31 (4 vCPU / 16GB) | €15 |
| DigitalOcean | 4 vCPU / 16GB (Sydney) | $48 |
| Vultr | 4 vCPU / 16GB (Sydney) | $48 |

After purchasing, SSH into the server.

### 2. Install Docker

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in for group change to take effect
```

### 3. Prepare the host

```bash
# Required for Elasticsearch
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 4. Clone and start RAGFlow

```bash
git clone https://github.com/infiniflow/ragflow.git
cd ragflow/docker

# Edit environment — change all passwords
nano .env

# Start all services
docker compose up -d
```

First start takes a few minutes. Check status:

```bash
docker compose ps        # all services should show "healthy"
docker logs -f ragflow-server  # watch startup logs
```

Verify it works locally:

```bash
curl -s http://localhost | head -5
```

### 5. Set up Cloudflare Tunnel

This exposes RAGFlow at `htm.localhub.nz` without opening any ports on the VPS.

#### a. Install cloudflared on the VPS

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
```

#### b. Authenticate with Cloudflare

```bash
cloudflared tunnel login
# This opens a URL — click it, select the localhub.nz domain, authorize
```

#### c. Create the tunnel

```bash
cloudflared tunnel create ragflow-htm
```

Note the tunnel ID printed (e.g. `a1b2c3d4-...`).

#### d. Create the config file

```bash
mkdir -p ~/.cloudflared
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <TUNNEL_ID>
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: htm.localhub.nz
    service: http://localhost:80
  - service: http_status:404
EOF
```

Replace `<TUNNEL_ID>` with the actual tunnel ID from step c.

#### e. Add DNS record

```bash
cloudflared tunnel route dns ragflow-htm htm.localhub.nz
```

This automatically creates a CNAME record in Cloudflare DNS.

#### f. Run the tunnel as a system service

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Now `https://htm.localhub.nz` should load the RAGFlow Web UI. SSL is handled by Cloudflare automatically.

### 6. Set up for HTM use

1. Open `https://htm.localhub.nz` in your browser
2. **Register** an account (first user becomes admin)
3. **Add an LLM provider** — Settings > Model Providers > add your API key (OpenAI, Azure OpenAI, or Ollama for local models)
4. **Create a Knowledge Base** — e.g. "Philips Monitors" or "GE Ventilators"
5. **Upload service manuals** — drag and drop PDFs, wait for parsing to complete
6. **Create an Assistant** — link it to your knowledge base, then start asking questions

## Knowledge Base Organization

In the RAGFlow Web UI, organize knowledge bases by manufacturer or device family:

| Knowledge Base | Example Contents |
|----------------|-----------------|
| Philips Monitors | MX800, IntelliVue MP Series service manuals |
| GE Ventilators | Carescape R860, CARESTATION 620 service manuals |
| Siemens Imaging | Artis zee service manual |
| General Reference | Electrical safety standards, common procedures |

Or use a single knowledge base if your manual collection is small.

## Project Structure

```
RAGflow-HTM/
├── README.md              # This file
├── docker/
│   └── .env.example       # Pre-tuned environment config
└── manuals/               # Local storage for your PDF manuals
```

RAGFlow itself is deployed from its own repository — we do not maintain a custom docker-compose.

## Useful Commands

```bash
# View RAGFlow logs (run from ragflow/docker/)
docker logs -f ragflow-server

# Restart RAGFlow
docker compose restart ragflow-server

# Stop all services
docker compose down

# Stop and remove all data (fresh start)
docker compose down -v

# Check Cloudflare Tunnel status
sudo systemctl status cloudflared
journalctl -u cloudflared -f
```

## Troubleshooting

**Services won't start?**
- Check `sysctl vm.max_map_count` (must be >= 262144)
- Check logs: `docker compose logs <service-name>`

**PDF not parsing correctly?**
- Try different chunking methods in Knowledge Base settings
- For scanned PDFs, ensure OCR is enabled in the parsing config
- Check that your LLM provider is configured (needed for embedding)

**htm.localhub.nz not loading?**
- Check tunnel: `sudo systemctl status cloudflared`
- Check RAGFlow: `docker compose ps` (all healthy?)
- Check DNS: `dig htm.localhub.nz` (should show CNAME to cfargotunnel.com)

**Out of memory?**
- Increase `MEM_LIMIT` in `.env`
- Consider upgrading VPS to 32GB RAM for larger manual collections
