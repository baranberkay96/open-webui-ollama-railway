# Open WebUI v0.12 + Ollama Private AI — Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Open WebUI v0.12 + Ollama Private AI** (version label `v0.12` = Open WebUI major.minor for a 0.x project; tags are `v0.x.y`) |
| Description | Your own private AI chat: run open models, add web search, RAG and multi-user. (78 chars) |
| Category | AI/ML |
| Tags | open-webui, ollama, llm, chatgpt-alternative, rag, searxng, pgvector, private-ai |
| User input at deploy | **`WEBUI_ADMIN_EMAIL`** (the admin login). Everything else is generated. |

## Base market template

- **`openwebui` "Open WebUI [with Postgres & Redis]"** (163 projects, 95 active, fastest-growing) — chosen as base: already uses Postgres, Redis websocket manager, pool settings and telemetry opt-outs. Fixed: unpinned image, **no volume** on Open WebUI (uploads, cache and the embedded Chroma vector store are lost on every redeploy), vectors not in Postgres, `RAILWAY_RUN_UID` unnecessary, no Ollama, no web search, open first-visitor admin registration.
- `2yb5lj` "Open WebUI" (1,106 projects, 438 active) — `:main` dev tag, points at `https://ollama.com`, SQLite on a volume. Kept only the volume path `/app/backend/data`.
- `ollama-openwebui-or-self-host-your-own-c` "Ollama + OpenWebUI" (232 projects) — unpinned images, hardcoded `WEBUI_SECRET_KEY=t0p-s3cr3t`, `CORS_ALLOW_ORIGIN=*`, no healthchecks, no model pre-pull. Kept the idea of Ollama on the private network with `OLLAMA_HOST=::`.
- No `_upstream` repos for this app. Market templates have no license → reference only.

## Pinned versions

| Service | Image / source | Tag | Release date | Checked |
|---|---|---|---|---|
| Open WebUI | `ghcr.io/open-webui/open-webui` | `v0.12.0` | 2026-10-10 | 2026-10-11 |
| Ollama | `ollama/ollama` (wrapped: `services/ollama`) | `0.40.3` | 2026-10-11 | 2026-10-11 |
| SearXNG | `searxng/searxng` (wrapped: `services/searxng`) | `2026.10.9-f4822b3fc` (= `latest` digest on 2026-10-11) | 2026-10-09 | 2026-10-11 |
| Postgres + pgvector | `pgvector/pgvector` | `0.8.7-pg18-trixie` | 2026-10-01 | 2026-10-11 |
| Redis | `redis` | `8.2` | Railway base | 2026-10-11 |
| Models (pulled at runtime) | `llama3.2:3b`, `nomic-embed-text:v1.5` | — | Ollama library | 2026-10-11 |

- `v0.12.0` is the current stable release (published the day before this check; release notes list no breaking changes, one index migration). Fallback if a regression appears: `v0.11.4`.
- Upgrade path: back up Postgres and the `/app/backend/data` volume, change the Open WebUI tag, redeploy (Alembic migrations run on start).
- Lighter variant: `ghcr.io/open-webui/open-webui:v0.12.0-slim` (no PyTorch; embeddings already come from Ollama, but local Whisper speech-to-text is not included).

## Architecture

```
Internet ──HTTPS──▶ Open WebUI :8080  (UI, API, websockets; volume /app/backend/data)
                      ├─▶ Ollama :11434   chat model llama3.2:3b + embeddings nomic-embed-text (volume /root/.ollama)
                      ├─▶ SearXNG :8080   private metasearch, JSON API for web search
                      ├─▶ Postgres (pgvector)  users, chats, settings + RAG vectors
                      └─▶ Redis           websocket manager, task control, shared app state
```

Upstream reference: Open WebUI `docker-compose.yaml` (Open WebUI + Ollama, volumes `/app/backend/data` and `/root/.ollama`) and its documented production options: PostgreSQL via `DATABASE_URL`, `VECTOR_DB=pgvector`, Redis via `REDIS_URL` + `WEBSOCKET_MANAGER=redis`, SearXNG web search, Ollama embeddings.

Key behaviours checked in the v0.12.0 source:

- **Admin bootstrap**: when `WEBUI_ADMIN_EMAIL` and `WEBUI_ADMIN_PASSWORD` are set and no user exists, Open WebUI creates that admin at startup and **turns sign-up off** (`main.py`). No first-visitor race.
- **pgvector**: Open WebUI runs `CREATE EXTENSION IF NOT EXISTS vector` itself and zero-pads embeddings up to `PGVECTOR_INITIALIZE_MAX_VECTOR_LENGTH`.
- **Workers**: migrations run at import in every Uvicorn worker without a lock, so the template starts with `UVICORN_WORKERS=1`; raise it after the first boot (Redis websocket manager is already configured for multi-worker).
- **Persistent config**: most settings are stored in the database after the first start (`ENABLE_PERSISTENT_CONFIG=true` default); later changes are made in *Admin Settings*, not by editing variables.

## Services

Create in this order. Values are entered exactly as shown (Railway resolves `${{...}}`).

### Open WebUI

- **Source:** Docker image `ghcr.io/open-webui/open-webui:v0.12.0`
- **Start command:** image default
- **Healthcheck:** `/health` (timeout 600 s)
- **Public domain:** yes → target port `8080`
- **TCP proxy:** none
- **Volume:** `/app/backend/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `8080` | HTTP port (public domain target). |
| `HOST` | `::` | Listen on IPv4+IPv6. |
| `WEBUI_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Public URL (links, OAuth callbacks, notifications). |
| `WEBUI_SECRET_KEY` | `${{secret(48, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Signs sessions/JWTs and encrypts OAuth tokens. Never change after first boot. |
| `WEBUI_ADMIN_EMAIL` | *(empty)* | **Input required.** Email (login) of the admin account created on first boot. Sign-up is disabled once it exists. |
| `WEBUI_ADMIN_PASSWORD` | `${{secret(20, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Password of the bootstrap admin account (generated). Change it after first login. |
| `WEBUI_ADMIN_NAME` | `Admin` | Display name of the bootstrap admin. |
| `DEFAULT_USER_ROLE` | `pending` | Role for users who sign up later (admin approves them). |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres for users, chats, settings and knowledge metadata. |
| `DATABASE_POOL_SIZE` | `10` | SQLAlchemy pool size per worker. |
| `DATABASE_POOL_MAX_OVERFLOW` | `5` | Extra connections above the pool size. |
| `DATABASE_POOL_RECYCLE` | `1800` | Recycle DB connections after N seconds. |
| `VECTOR_DB` | `pgvector` | Store RAG embeddings in Postgres (pgvector) instead of the embedded Chroma. |
| `PGVECTOR_INITIALIZE_MAX_VECTOR_LENGTH` | `1536` | Vector column size; fits nomic-embed-text (768, zero-padded) and 1536-dim API embeddings. Fixed after first use. |
| `REDIS_URL` | `${{Redis.REDIS_URL}}` | Redis for app state, task control and websockets. |
| `WEBSOCKET_MANAGER` | `redis` | Socket.IO state in Redis (required for more than one worker). |
| `ENABLE_WEBSOCKET_SUPPORT` | `true` | Real-time streaming over websockets. |
| `UVICORN_WORKERS` | `1` | Uvicorn worker processes. Raise to 2-4 after the first successful boot (migrations run per worker at startup). |
| `OLLAMA_BASE_URL` | `http://${{Ollama.RAILWAY_PRIVATE_DOMAIN}}:11434` | Private Ollama endpoint. |
| `DEFAULT_MODELS` | `llama3.2:3b` | Model preselected for new chats. |
| `RAG_EMBEDDING_ENGINE` | `ollama` | Compute document embeddings with Ollama (no local embedding model in this container). |
| `RAG_EMBEDDING_MODEL` | `nomic-embed-text:v1.5` | Embedding model pulled by the Ollama service. |
| `ENABLE_WEB_SEARCH` | `true` | Enable web search in chats. |
| `WEB_SEARCH_ENGINE` | `searxng` | Use the private SearXNG service. |
| `SEARXNG_QUERY_URL` | `http://${{SearXNG.RAILWAY_PRIVATE_DOMAIN}}:8080/search` | SearXNG search endpoint (JSON format). |
| `WEB_SEARCH_RESULT_COUNT` | `5` | Search results fetched per query. |
| `OPENAI_API_KEY` | *(empty)* | Optional: add an OpenAI (or compatible) key to use hosted models alongside local ones. |
| `CORS_ALLOW_ORIGIN` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Allowed browser origins (semicolon-separated). |
| `ANONYMIZED_TELEMETRY` | `false` | Disable anonymous telemetry. |
| `DO_NOT_TRACK` | `true` | Disable tracking. |
| `SCARF_NO_ANALYTICS` | `true` | Disable Scarf analytics. |

### Ollama

- **Source:** GitHub repo `baranberkay96/open-webui-ollama-railway`, root directory `/services/ollama` (Dockerfile build)
- **Start command:** image default
- **Healthcheck:** `/` (timeout 300 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/root/.ollama`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `11434` | Ollama API port (private). |
| `OLLAMA_HOST` | `[::]:11434` | Bind address for the Ollama API (IPv4+IPv6). |
| `OLLAMA_PULL_MODELS` | `llama3.2:3b,nomic-embed-text:v1.5` | Models pulled on first boot (comma-separated): a chat model and the RAG embedding model. |
| `OLLAMA_CONTEXT_LENGTH` | `8192` | Default context window for chats with documents/web results. |
| `OLLAMA_KEEP_ALIVE` | `5m` | Unload idle models after this time to free RAM. |
| `OLLAMA_MAX_LOADED_MODELS` | `2` | Keep chat + embedding model loaded together. |

### SearXNG

- **Source:** GitHub repo `baranberkay96/open-webui-ollama-railway`, root directory `/services/searxng` (Dockerfile build)
- **Start command:** image default
- **Healthcheck:** `/healthz` (timeout 120 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `8080` | SearXNG port (private). |
| `SEARXNG_SECRET` | `${{secret(48, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | SearXNG server secret. |
| `SEARXNG_BASE_URL` | `http://${{RAILWAY_PRIVATE_DOMAIN}}:8080/` | Internal base URL (instance is private-only). |

### Postgres

- **Source:** Docker image `pgvector/pgvector:0.8.7-pg18-trixie`
- **Start command:** image default
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/var/lib/postgresql/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1
- **Note:** Variables and volume copied unchanged from templates/_railway-base/postgres.json; image swapped to the official pgvector build (PostgreSQL 18 + pgvector 0.8.7) because Open WebUI stores RAG vectors here. SSL_CERT_DAYS is unused by this image.

| Variable | Value | Description |
|---|---|---|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Location where the database will be initialized |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Railway Private Domain Name. |
| `PGPORT` | `5432` | Port to connect to Postgres. |
| `PGUSER` | `${{ POSTGRES_USER }}` | Required variable for Data panel |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Required variable for the data panel. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Required variable for Data panel |
| `POSTGRES_DB` | `railway` | Default database created when image is started. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | URL to connect to Postgres database. |
| `POSTGRES_USER` | `postgres` | User to connect to Postgres DB |
| `SSL_CERT_DAYS` | `820` | SSL certificate expiry in days. |
| `POSTGRES_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Password to connect to DB |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Allow Postgres to cleanly shut down |

### Redis

- **Source:** Docker image `redis:8.2`
- **Start command:** `/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH"`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1
- **Note:** Copied unchanged from templates/_railway-base/redis.json

| Variable | Value | Description |
|---|---|---|
| `REDISHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Private network hostname of the Redis service, only resolvable from services in the same environment |
| `REDISPORT` | `6379` | Port that Redis listens on |
| `REDISUSER` | `default` | Username for authenticating with Redis |
| `REDIS_URL` | `redis://${{ REDISUSER }}:${{ REDIS_PASSWORD }}@${{ REDISHOST }}:${{ REDISPORT }}` | Connection string for connecting to Redis using the private network |
| `REDISPASSWORD` | `${{REDIS_PASSWORD}}` | Alias of REDIS_PASSWORD for clients that expect the unseparated name |
| `REDIS_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Randomly generated password for authenticating with Redis |

## Improvements over market templates

1. **Pinned Open WebUI `v0.12.0`** instead of `:main`/untagged images; Ollama, SearXNG and pgvector pinned too.
2. **Persistent volume** on `/app/backend/data` (uploads, caches) — missing in the Postgres/Redis market template.
3. **RAG vectors in Postgres (pgvector)** instead of the embedded Chroma files; embeddings from Ollama (`nomic-embed-text`), so no model download inside the web container.
4. **Web search built in**: private SearXNG with JSON output enabled, wired via `SEARXNG_QUERY_URL`.
5. **Ollama included** with automatic first-boot pull of a chat and an embedding model, volume-backed; private network only.
6. **Safe admin bootstrap**: admin created from `WEBUI_ADMIN_EMAIL` + generated password, sign-up then disabled; later sign-ups default to `pending`.
7. **Secrets generated** (`WEBUI_SECRET_KEY`, SearXNG secret) instead of `t0p-s3cr3t`; CORS limited to the app's own origin.
8. Health checks on every HTTP service (`/health`, `/`, `/healthz`); Postgres/Redis not exposed publicly.

## Scaling

- **Open WebUI**: raise `UVICORN_WORKERS` (2–4) after the first boot; each worker adds ~0.4–0.6 GB RAM. Replicas are not possible while the data volume is attached.
- **Ollama**: scale vertically. RAM ≈ model file size + context (3B ≈ 3 GB, 7–8B ≈ 6–8 GB). CPU-only on Railway; larger models answer slowly — add an OpenAI-compatible API key for heavy use.
- **SearXNG**: stateless; one instance is enough for a team.

## Resource estimate

Prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month.

| Service | Idle RAM | Typical RAM | Typical vCPU | Volume |
|---|---|---|---|---|
| Open WebUI (1 worker) | 450 MB | 800 MB | 0.02 / 0.15 | ~1 GB |
| Ollama (llama3.2:3b + embeddings) | 60 MB (models unloaded) | 3.2 GB | 0 / 0.6 avg (2–4 vCPU bursts while generating) | ~3 GB |
| SearXNG | 120 MB | 200 MB | 0.01 / 0.03 | — |
| Postgres + pgvector | 80 MB | 300 MB | 0.02 / 0.05 | ~1 GB |
| Redis | 10 MB | 30 MB | 0.01 | ~0.2 GB |
| **Total** | **≈ 0.72 GB** | **≈ 4.5 GB** | **≈ 0.05 / 0.85** | **≈ 5.2 GB** |

- **Idle:** 0.72 × $10 + 0.05 × $20 + 5.2 × $0.15 ≈ **$9/month**
- **Typical (small team chatting with the local model daily):** 4.5 × $10 + 0.85 × $20 + $0.8 ≈ **$63/month**
- Using only hosted models through `OPENAI_API_KEY` (Ollama idle): ≈ $15–20/month.
- **Minimum plan: Hobby** (Ollama with a 3B model needs ~3 GB RAM; Trial's 1 GB per service is too small). Hobby's 5 GB volume per service fits the two default models; bigger model collections need Pro.

## Verification checklist

1. Healthchecks green: Open WebUI `/health`, Ollama `/` ("Ollama is running"), SearXNG `/healthz`.
2. Ollama logs show `llama3.2:3b` and `nomic-embed-text:v1.5` pulled (first boot downloads ~2.3 GB).
3. Open the public URL and sign in with `WEBUI_ADMIN_EMAIL` / `WEBUI_ADMIN_PASSWORD` (Open WebUI service variables); the sign-up link is absent. Change the password under *Settings → Account*.
4. New chat with `llama3.2:3b` → streamed answer.
5. Turn on **Web Search** in a chat and ask about a current event → answer cites web sources (SearXNG JSON API reachable).
6. Upload a PDF to a chat or *Workspace → Knowledge* and ask about it → answer uses the document (embeddings via Ollama, vectors stored in Postgres: table `document_chunk` exists).
7. *Admin Panel → Users*: create a second user; log in from another browser; both see real-time updates.
8. Redeploy all services → users, chats, uploaded files, knowledge bases and models persist.

## Open questions

- **Not deployed yet**: verified against upstream source for Open WebUI `v0.12.0`, Ollama `v0.40.3`, SearXNG `2026.10.9`.
- **Brand-new release**: `v0.12.0` was one day old when pinned; if a regression shows up, pin `v0.11.4` (same configuration).
- **SearXNG engines** from Railway datacenter IPs: some engines (e.g. Google) may rate-limit or block; others keep returning results. Tune engines in `services/searxng/settings.yml`.
- **Duplicate admin race** only if `UVICORN_WORKERS` > 1 on the very first boot; the template ships with 1.
- **pgvector dimension** is fixed at first use (1536 with zero padding). Switching to an embedding model with more than 1536 dimensions requires raising `PGVECTOR_INITIALIZE_MAX_VECTOR_LENGTH` before any document is indexed (or re-indexing).
- **Open WebUI license**: Open WebUI is under its own BSD-3-based license with a branding clause (keep the "Open WebUI" branding unless you qualify for an exception or hold an enterprise license). Mentioned in README.

## Deploy test findings (2026-10-11)

Not yet deploy-tested, but `HOST=::` with Open WebUI's single uvicorn worker is IPv6-only (asyncio sets IPV6_V6ONLY); the same pattern failed Dify's agent backend healthcheck. Fix applied pre-emptively: `HOST=0.0.0.0`.
