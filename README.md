# RAGFlow-HTM

AI-powered service manual knowledge base for Healthcare Technology Management (HTM) engineers.

Upload device service manuals (PDF, Word, Excel, scanned images) and ask natural language questions like:

> "Philips MX800 error code 42 是什么意思？"
> "What is the PM schedule for GE Carescape R860?"
> "Siemens Artis zee troubleshooting steps for error E204"

Built on [RAGFlow](https://github.com/infiniflow/ragflow) — an open-source RAG engine with deep document understanding.

## Features

- **Deep PDF parsing** — handles complex layouts, tables, multi-column text, headers/footers
- **OCR support** — works with scanned service manuals
- **Multi-format** — PDF, Word, Excel, PowerPoint, images
- **Web UI** — built-in chat interface for engineers to query manuals
- **REST API** — integrate with CMMS/EAM systems (port 9380)
- **Docker deployment** — single command to start all services

## System Requirements

| Resource | Minimum |
|----------|---------|
| CPU | 4 cores |
| RAM | 16 GB |
| Disk | 50 GB |
| Docker | 24.0.0+ |
| Docker Compose | v2.26.1+ |

## Quick Start

### 1. Clone and configure

```bash
git clone https://github.com/HugonzAI/RAGflow-HTM.git
cd RAGflow-HTM/docker

# Create your environment config
cp .env.example .env

# IMPORTANT: Edit .env and change all default passwords
nano .env
```

### 2. Prepare the host

```bash
# Required kernel tuning for Elasticsearch
sudo sysctl -w vm.max_map_count=262144

# Make it persistent across reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 3. Start the services

```bash
cd docker
docker compose up -d
```

Wait for all services to become healthy (this may take a few minutes on first start):

```bash
docker compose ps
```

### 4. Access the Web UI

Open your browser and navigate to:

- **Web UI**: `http://<your-server-ip>` (port 80)
- **API**: `http://<your-server-ip>:9380`
- **MinIO Console**: `http://<your-server-ip>:9001` (for storage management)

### 5. Initial Setup in the Web UI

1. **Register** an admin account (first user becomes admin)
2. **Configure an LLM provider** — Go to Settings → Model Providers and add your LLM API key (OpenAI, Azure OpenAI, Ollama for local models, etc.)
3. **Create a Knowledge Base** — Name it something like "Service Manuals" or organize by manufacturer
4. **Upload PDFs** — Upload your device service manuals
5. **Create an Assistant** — Link it to the knowledge base and start chatting

## Recommended Knowledge Base Organization

For HTM use, consider organizing knowledge bases by:

```
├── Philips/
│   ├── MX800 Service Manual.pdf
│   ├── IntelliVue MP Series.pdf
│   └── ...
├── GE Healthcare/
│   ├── Carescape R860.pdf
│   ├── CARESTATION 620-650.pdf
│   └── ...
├── Siemens/
│   ├── Artis zee Service Manual.pdf
│   └── ...
└── General/
    ├── Electrical Safety Standards.pdf
    └── ...
```

Or create one knowledge base per device family for more focused retrieval.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  HTM Engineer                    │
│              (Browser / API Client)              │
└──────────────────┬──────────────────────────────┘
                   │
         ┌─────────▼─────────┐
         │     RAGFlow        │  :80 (Web UI)
         │   (Application)    │  :9380 (API)
         └──┬──┬──┬──┬───────┘
            │  │  │  │
   ┌────────┘  │  │  └────────┐
   ▼           ▼  ▼           ▼
┌──────┐ ┌─────┐ ┌─────┐ ┌──────┐
│ ES   │ │MySQL│ │MinIO│ │Redis │
│:1200 │ │:5455│ │:9000│ │:6379 │
└──────┘ └─────┘ └─────┘ └──────┘
```

| Service | Purpose |
|---------|---------|
| **Elasticsearch** | Full-text search and vector storage for document chunks |
| **MySQL** | Metadata, user accounts, knowledge base configuration |
| **MinIO** | Object storage for uploaded documents (PDFs, images) |
| **Redis** | Caching and session management |

## Configuration

All configuration is in `docker/.env`. Key settings:

| Variable | Default | Description |
|----------|---------|-------------|
| `RAGFLOW_IMAGE` | `infiniflow/ragflow:v0.24.0` | RAGFlow Docker image |
| `DOC_ENGINE` | `elasticsearch` | Search engine backend |
| `DEVICE` | `cpu` | `cpu` or `gpu` (NVIDIA) |
| `DOC_BULK_SIZE` | `4` | Document processing batch size |
| `EMBEDDING_BATCH_SIZE` | `16` | Embedding computation batch size |

## Useful Commands

```bash
# View logs
docker compose logs -f ragflow

# Restart a specific service
docker compose restart ragflow

# Stop all services
docker compose down

# Stop and remove all data (fresh start)
docker compose down -v

# Check service health
docker compose ps
```

## Troubleshooting

**Services fail to start?**
- Check `vm.max_map_count`: `sysctl vm.max_map_count` (must be ≥ 262144)
- Ensure ports 80, 443, 9380, 1200, 5455, 9000, 9001, 6379 are available
- Check logs: `docker compose logs <service-name>`

**PDF not parsing correctly?**
- Try different chunking templates in the Knowledge Base settings
- For scanned PDFs, ensure OCR is enabled
- Increase `DOC_BULK_SIZE` if processing is slow

**Out of memory?**
- Increase `MEM_LIMIT` in `.env`
- Increase Docker's memory allocation
- Consider using `DEVICE=gpu` for embedding acceleration

## License

This project scaffold is MIT licensed. RAGFlow itself is licensed under the [Apache 2.0 License](https://github.com/infiniflow/ragflow/blob/main/LICENSE).
