# Notice

## Upstream software

| Software | License | Images used (publisher) |
|---|---|---|
| Open WebUI | Open WebUI License (BSD-3-Clause-based with an additional branding clause) — Copyright (c) 2023- Open WebUI Inc. | `ghcr.io/open-webui/open-webui:v0.12.0` (Open WebUI) |
| Ollama | MIT — Copyright (c) Ollama | `ollama/ollama:0.40.3` (Ollama) |
| SearXNG | AGPL-3.0 | `searxng/searxng:2026.10.9-f4822b3fc` (SearXNG) |
| PostgreSQL + pgvector | PostgreSQL License | `pgvector/pgvector:0.8.7-pg18-trixie` (pgvector) |
| Redis | RSALv2 / SSPLv1 / AGPLv3 (Redis 8 tri-license) | `redis:8.2` (Docker Official Image) |
| Models pulled at runtime | Llama 3.2 Community License (`llama3.2:3b`, Meta); Apache-2.0 (`nomic-embed-text:v1.5`, Nomic AI) | Ollama library |

## Verified against

| What | Upstream file | Ref | URL | Checked |
|---|---|---|---|---|
| Release / image tags (incl. `-slim`, `-ollama`) | GitHub releases, GHCR manifests | `v0.12.0` | https://github.com/open-webui/open-webui/releases/tag/v0.12.0 | 2026-10-11 |
| Reference compose (Open WebUI + Ollama, volumes) | `docker-compose.yaml` | `v0.12.0` | https://github.com/open-webui/open-webui/blob/v0.12.0/docker-compose.yaml | 2026-10-11 |
| Image defaults (root user, `HF_HOME` under data dir, slim build) | `Dockerfile` | `v0.12.0` | https://github.com/open-webui/open-webui/blob/v0.12.0/Dockerfile | 2026-10-11 |
| Start script (`HOST`, `PORT`, `UVICORN_WORKERS`, secret key file) | `backend/start.sh` | `v0.12.0` | https://github.com/open-webui/open-webui/blob/v0.12.0/backend/start.sh | 2026-10-11 |
| Env names/defaults (DB pool, Redis, websocket, admin bootstrap, workers, migrations) | `backend/open_webui/env.py` | `v0.12.0` | https://github.com/open-webui/open-webui/blob/v0.12.0/backend/open_webui/env.py | 2026-10-11 |
| Persistent config, vector DB, RAG engine, web search, CORS, default models | `backend/open_webui/config.py` | `v0.12.0` | https://github.com/open-webui/open-webui/blob/v0.12.0/backend/open_webui/config.py | 2026-10-11 |
| Admin creation from env + sign-up disabled | `backend/open_webui/main.py`, `backend/open_webui/utils/auth.py` | `v0.12.0` | https://github.com/open-webui/open-webui/tree/v0.12.0/backend/open_webui | 2026-10-11 |
| pgvector extension creation, padding, table name | `backend/open_webui/retrieval/vector/dbs/pgvector.py` | `v0.12.0` | same | 2026-10-11 |
| SearXNG query URL handling (`format=json`) | `backend/open_webui/retrieval/web/searxng.py` | `v0.12.0` | same | 2026-10-11 |
| SearXNG container (Granian on `::`:8080, config path, entrypoint ownership) | `container/entrypoint.sh`, `container/dist.dockerfile`, `container/.env.example` | `master` (image `2026.10.9-f4822b3fc`) | https://github.com/searxng/searxng/tree/master/container | 2026-10-11 |
| SearXNG `SEARXNG_SECRET`, `formats`, `/healthz` | `searx/settings.yml`, `searx/webapp.py` | `master` | https://github.com/searxng/searxng/tree/master/searx | 2026-10-11 |
| Ollama release | GitHub releases, Docker Hub tag | `v0.40.3` | https://github.com/ollama/ollama/releases/tag/v0.40.3 | 2026-10-11 |

## Adapted code

| Our file | Source | Copyright / license | Changes |
|---|---|---|---|
| `services/ollama/railway-entrypoint.sh` | Idea of the `ollama-pull-llama` init container in `n8n-io/self-hosted-ai-starter-kit` | Copyright 2024-present n8n GmbH — Apache License 2.0 | Re-implemented as an in-container wrapper (Railway volumes cannot be shared with an init container); multiple models; skips models already present |
| `services/searxng/settings.yml`, `services/searxng/Dockerfile`, `services/ollama/Dockerfile` | Written for this template from the SearXNG and Ollama documentation | — | — |

## Reference only (not copied)

- Marketplace templates `openwebui` ("Open WebUI [with Postgres & Redis]"), `2yb5lj` ("Open WebUI"), `ollama-openwebui-or-self-host-your-own-c` ("Ollama + OpenWebUI"): no license; reviewed for structure (volume path, Postgres/Redis variables, Ollama on the private network). No files or text copied; secrets regenerated.
- No `_upstream` repositories were provided for this app.

## Trademarks

"Open WebUI" and its logo belong to Open WebUI Inc. and must remain visible per its license; "Ollama", "SearXNG", "Llama" and "Railway" belong to their respective owners. This is a community template and is not affiliated with or endorsed by them.
