# Deploy and Host Open WebUI with Railway

Open WebUI is a self-hosted chat interface for large language models, similar to ChatGPT, that works with local models through Ollama and with any OpenAI-compatible API. This community template runs Open WebUI with Ollama, private SearXNG web search, Postgres with pgvector for RAG, and Redis.

## About Hosting Open WebUI

Open WebUI on its own is a single container, but a private, multi-user deployment needs several supporting pieces: a model server for open models, a database that survives redeploys, a vector store for document chat, a search backend for web-grounded answers, and Redis for real-time connections across workers. This template provides all of them on Railway's private network. Ollama pulls a small chat model and an embedding model on first boot, SearXNG answers web searches without third-party search keys, and pgvector keeps embeddings in Postgres. The admin account is created from variables at startup and public sign-up is closed.

## Common Use Cases

- A private ChatGPT-style assistant for a team, running open models without sending prompts to third parties
- Chat with internal documents and knowledge bases (RAG) stored in your own database
- Web-grounded answers through a private metasearch engine
- One interface for local models plus hosted APIs (OpenAI-compatible keys) with per-user access control
- Evaluating and comparing open models before choosing a production LLM

## Dependencies for Open WebUI Hosting

- Open WebUI (`ghcr.io/open-webui/open-webui`)
- Ollama (local models and embeddings)
- SearXNG (web search)
- PostgreSQL with pgvector (application data and RAG vectors)
- Redis (websocket manager and shared state)

### Deployment Dependencies

- Open WebUI repository: https://github.com/open-webui/open-webui
- Open WebUI documentation: https://docs.openwebui.com/
- Environment variable reference: https://docs.openwebui.com/reference/env-configuration/
- Ollama: https://github.com/ollama/ollama
- SearXNG Docker installation: https://docs.searxng.org/admin/installation-docker.html
- pgvector: https://github.com/pgvector/pgvector

### Implementation Details

| Service | Image | Role | Public |
|---|---|---|---|
| Open WebUI | `ghcr.io/open-webui/open-webui:v0.12.0` | Chat UI and API; volume `/app/backend/data` for uploads and caches | **yes** (port 8080) |
| Ollama | `ollama/ollama:0.40.3` (repo build) | Runs `llama3.2:3b` and `nomic-embed-text:v1.5`, pulled on first boot; volume `/root/.ollama` | no |
| SearXNG | `searxng/searxng:2026.10.9-f4822b3fc` (repo build) | Private metasearch with JSON API for web search | no |
| Postgres | `pgvector/pgvector:0.8.7-pg18-trixie` | Users, chats, settings and RAG vectors | no |
| Redis | Railway Redis 8.2 | Websocket manager, task control | no |

**First login**

1. When deploying, enter your email in `WEBUI_ADMIN_EMAIL`.
2. After the first deploy, copy `WEBUI_ADMIN_PASSWORD` from the Open WebUI service variables and sign in at the public URL. Change the password under *Settings → Account*.
3. Wait for the Ollama logs to show both models pulled (about 2.3 GB on first boot), then start a chat with `llama3.2:3b`.
4. Add users in *Admin Panel → Users*, or re-enable sign-up there; new sign-ups stay *pending* until approved.

**Note on settings:** Open WebUI saves most settings in the database after the first start. Later changes are made in *Admin Settings*; editing the corresponding variables has no effect unless `ENABLE_PERSISTENT_CONFIG=false`.

**Models:** pull more models from *Admin Settings → Models* or by extending `OLLAMA_PULL_MODELS`. Railway has no GPUs, so local models run on CPU; keep them small, or add an OpenAI-compatible key (`OPENAI_API_KEY` or *Admin Settings → Connections*) for larger hosted models.

**Scaling workers:** raise `UVICORN_WORKERS` on Open WebUI to 2–4 after the first successful start (Redis is already configured for multiple workers). Give Ollama more RAM for larger models.

**Pinning and upgrading:** every image is pinned. To upgrade Open WebUI, back up Postgres and the data volume, change the tag and redeploy; database migrations run on start.

**License note:** Open WebUI uses the Open WebUI License (BSD-3-based with a branding clause): its name and logo must stay visible unless you have 50 or fewer end users in a 30-day period, written permission, or an enterprise license.

### Why Deploy Open WebUI on Railway?

Railway gives Open WebUI, its model server, search engine and databases a private network, managed HTTPS and persistent volumes in one project, so your chats, documents and models stay under your control without managing servers. Resources scale from the dashboard and every deploy can be rolled back. This is a community template and is not affiliated with Open WebUI Inc., Ollama or SearXNG.
