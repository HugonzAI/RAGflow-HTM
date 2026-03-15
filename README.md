# RAGFlow-HTM

AI-powered service manual knowledge base for Healthcare Technology Management (HTM) engineers.

Upload device service manuals (PDF, Word, Excel, scanned images) and ask natural language questions like:

> "Philips MX800 error code 42 是什么意思？"
> "What is the PM schedule for GE Carescape R860?"
> "Siemens Artis zee troubleshooting steps for error E204"

Built on [RAGFlow](https://github.com/infiniflow/ragflow) — an open-source RAG engine with deep document understanding.

## System Requirements

| Resource | Minimum |
|----------|---------|
| CPU | 4 cores |
| RAM | 16 GB |
| Disk | 50 GB |
| Docker | 24.0.0+ |
| Docker Compose | v2.26.1+ |

## Quick Start

### 1. Prepare the host

```bash
# Required for Elasticsearch
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 2. Clone RAGFlow

```bash
git clone https://github.com/infiniflow/ragflow.git
cd ragflow/docker
```

### 3. Configure environment

Use our pre-tuned `.env.example` as a starting point, or edit RAGFlow's default `.env` directly:

```bash
# Option A: use our template
cp /path/to/RAGflow-HTM/docker/.env.example .env

# Option B: edit RAGFlow's default .env in place
nano .env
```

Either way, **change all passwords** before starting.

### 4. Start services

```bash
docker compose up -d
```

First start may take a few minutes. Check status with:

```bash
docker compose ps        # all services should show "healthy"
docker logs -f ragflow-server  # watch RAGFlow startup logs
```

### 5. Access the Web UI

- **Web UI**: `http://<your-server-ip>` (port 80)
- **API**: `http://<your-server-ip>:9380`

### 6. Set up for HTM use

1. **Register** an account (first user becomes admin)
2. **Add an LLM provider** — Settings > Model Providers > add your API key (OpenAI, Azure OpenAI, or Ollama for local models)
3. **Create a Knowledge Base** — e.g. "Philips Monitors" or "GE Ventilators"
4. **Upload service manuals** — drag and drop PDFs, wait for parsing to complete
5. **Create an Assistant** — link it to your knowledge base, then start asking questions

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
# View RAGFlow logs
docker logs -f ragflow-server

# Restart RAGFlow
docker compose restart ragflow-server

# Stop all services
docker compose down

# Stop and remove all data (fresh start)
docker compose down -v
```

## Troubleshooting

**Services won't start?**
- Check `sysctl vm.max_map_count` (must be >= 262144)
- Check port availability: 80, 443, 9380
- Check logs: `docker compose logs <service-name>`

**PDF not parsing correctly?**
- Try different chunking methods in Knowledge Base settings
- For scanned PDFs, ensure OCR is enabled in the parsing config
- Check that your LLM provider is configured (needed for embedding)

**Out of memory?**
- Increase `MEM_LIMIT` in `.env`
- Increase Docker's memory allocation in Docker Desktop settings
