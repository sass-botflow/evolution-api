# BotFlow Evolution API — Production (Multi-Tenant SaaS)

Production configuration for **Evolution API v2.3.7** on BotFlow (`botflow.ink`), Qunvert-style: one WhatsApp session per workspace tenant, QR via authenticated backend only, global webhooks to `api.botflow.ink`.

---

## Architecture

```
Tenant (browser, JWT)
    │
    ▼
www.botflow.ink (Next.js BFF)
    │  /api/whatsapp/sessions/:id/qr  ← JWT required, QR never public
    ▼
api.botflow.ink (NestJS)
    │  POST /instance/create          ← apikey header
    │  POST /webhooks/evolution       ← Evolution events
    ▼
evolution.api.botflow.ink (Evolution API)
    ├── Redis (session cache, reconnect)
    └── PostgreSQL evolution DB (instances, history)
```

**Instance naming:** `botflow-{workspace8}-{random4}` — one session per workspace (multi-tenant isolation).

---

## 1. Docker Compose

File: `docker-compose.yml` (this repo root)

Services:
| Service | Role |
|---------|------|
| `evolution-db-init` | Creates `evolution` database + `evolution_api` schema |
| `evolution-redis` | Session cache, auto-restore on restart |
| `evolution-api` | `evoapicloud/evolution-api:v2.3.7` |

**EasyPanel deploy:**
1. http://187.124.12.89:3000 → `sass-botflow` → Compose service
2. Source → paste `docker-compose.yml` from this repo
3. **Environment tab: EMPTY** (all vars hardcoded in compose)
4. Domain: `evolution.api.botflow.ink` → port `8080`
5. Deploy (wait 2–3 min)

---

## 2. Environment Variables

Full reference: `production.env`

### Evolution (in compose — do not duplicate in EasyPanel)

| Variable | Value |
|----------|-------|
| `SERVER_URL` | `https://evolution.api.botflow.ink` |
| `AUTHENTICATION_API_KEY` | `e323292ec09f14a66514089b993e246eb469f3a04448ddc02167447e76621f18` |
| `DATABASE_PROVIDER` | `postgresql` |
| `DATABASE_CONNECTION_URI` | `postgresql://botflow:botflow@sass-botflow_postgres:5432/evolution?schema=evolution_api&connection_limit=25&pool_timeout=30` |
| `REDIS_URI` / `CACHE_REDIS_URI` | `redis://evolution-redis:6379/1` |
| `WEBSOCKET_ENABLED` | `true` |
| `WEBHOOK_GLOBAL_ENABLED` | `true` |
| `WEBHOOK_GLOBAL_URL` | `https://api.botflow.ink/webhooks/evolution` |

### Webhook events enabled

| User event | Evolution env var |
|------------|-------------------|
| `QRCODE_UPDATED` | `WEBHOOK_EVENTS_QRCODE_UPDATED=true` |
| `CONNECTION_UPDATE` | `WEBHOOK_EVENTS_CONNECTION_UPDATE=true` |
| `MESSAGES_UPSERT` / `MESSAGE_RECEIVED` | `WEBHOOK_EVENTS_MESSAGES_UPSERT=true` |
| `SEND_MESSAGE` | `WEBHOOK_EVENTS_SEND_MESSAGE=true` |
| Typing / presence | `WEBHOOK_EVENTS_PRESENCE_UPDATE=true` |
| Read receipts | `readMessages=true` + `readStatus=true` on instance create |

### Backend (EasyPanel → backend service)

```env
EVOLUTION_API_URL=http://sass-botflow_evolution-api:8080
EVOLUTION_API_KEY=e323292ec09f14a66514089b993e246eb469f3a04448ddc02167447e76621f18
EVOLUTION_WEBHOOK_SECRET=e323292ec09f14a66514089b993e246eb469f3a04448ddc02167447e76621f18
```

`EVOLUTION_WEBHOOK_SECRET` validates `apikey` header on incoming webhooks (same value as API key).

### JWT (tenant auth — not Evolution)

BotFlow uses **JWT** on all `/api/whatsapp/*` routes. QR codes are served only through JWT-authenticated BFF → backend. Evolution manager UI is disabled (`SERVER_DISABLE_MANAGER=true`).

---

## 3. Reverse Proxy

### EasyPanel / Traefik (current setup)

| Domain | Service | Port | TLS |
|--------|---------|------|-----|
| `evolution.api.botflow.ink` | evolution-api | 8080 | Cloudflare → EasyPanel Traefik |
| `api.botflow.ink` | backend | 8000 | Cloudflare |
| `www.botflow.ink` | frontend | 3000 | Cloudflare |

### Cloudflare DNS

```
evolution.api.botflow.ink  A  187.124.12.89  (proxied)
api.botflow.ink            A  187.124.12.89  (proxied)
www.botflow.ink            A  187.124.12.89  (proxied)
```

### Traefik labels (if manual compose)

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.evolution.rule=Host(`evolution.api.botflow.ink`)
  - traefik.http.routers.evolution.entrypoints=websecure
  - traefik.http.routers.evolution.tls=true
  - traefik.http.services.evolution.loadbalancer.server.port=8080
```

### Security at proxy layer

- **Do not** expose Evolution manager (`SERVER_DISABLE_MANAGER=true`)
- Restrict Evolution to Cloudflare IP ranges if direct access is a concern
- Webhook endpoint `POST /webhooks/evolution` must be publicly reachable by Evolution container (outbound HTTPS)

---

## 4. Health Endpoint

| Endpoint | Expected |
|----------|----------|
| `GET https://evolution.api.botflow.ink/health` | HTTP 200 JSON |
| `GET https://api.botflow.ink/health` | `config.evolution: true` |
| `GET https://api.botflow.ink/health/evolution` | DNS/TCP/HTTP probe to Evolution |
| `GET https://www.botflow.ink/api/setup-status` | `ready: true` |

Docker healthcheck (in compose):

```yaml
healthcheck:
  test: ['CMD', 'curl', '-f', 'http://127.0.0.1:8080/health']
  interval: 30s
  start_period: 120s
```

Verify after deploy:

```bash
curl -s https://evolution.api.botflow.ink/health
curl -s https://api.botflow.ink/health/evolution | jq .
curl -s https://www.botflow.ink/api/setup-status | jq '.ready'
```

---

## 5. Backup Strategy

### PostgreSQL (`evolution` database)

Daily cron on server:

```bash
#!/bin/bash
BACKUP_DIR=/var/backups/botflow
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p "$BACKUP_DIR"
docker exec sass-botflow_postgres pg_dump -U botflow -d evolution -Fc \
  -f /tmp/evolution_${DATE}.dump
docker cp sass-botflow_postgres:/tmp/evolution_${DATE}.dump \
  "$BACKUP_DIR/evolution_${DATE}.dump"
# Retain 14 days
find "$BACKUP_DIR" -name 'evolution_*.dump' -mtime +14 -delete
```

Restore:

```bash
docker cp evolution_YYYYMMDD.dump sass-botflow_postgres:/tmp/restore.dump
docker exec sass-botflow_postgres pg_restore -U botflow -d evolution -c /tmp/restore.dump
```

### Redis (Evolution cache)

```bash
docker exec botflow_evolution_redis redis-cli BGSAVE
docker cp botflow_evolution_redis:/data/dump.rdb /var/backups/botflow/redis_evolution.rdb
```

Redis is **cache** — safe to rebuild; Postgres + instance volume are authoritative.

### Instance sessions (Baileys auth)

```bash
docker run --rm -v botflow_evolution_instances:/data -v /var/backups/botflow:/backup \
  alpine tar czf /backup/evolution_instances_$(date +%Y%m%d).tar.gz -C /data .
```

Restore: extract back into `botflow_evolution_instances` volume, restart `evolution-api`.

### RPO / RTO targets

| Asset | RPO | RTO |
|-------|-----|-----|
| Postgres evolution | 24h (daily dump) | 15 min |
| Instance volume | 24h | 10 min |
| Redis cache | N/A (rebuild) | 2 min |

---

## 6. Security Checklist

- [x] `AUTHENTICATION_API_KEY` set (64-char hex) — never commit to public repos in future rotations
- [x] `AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES=false`
- [x] `SERVER_DISABLE_MANAGER=true` — no public admin UI
- [x] `SERVER_DISABLE_DOCS=true` — no Swagger on Evolution
- [x] `CORS_ORIGIN=https://api.botflow.ink` — browser cannot call Evolution directly
- [x] QR via webhook → Redis cache → JWT-only `/api/whatsapp/sessions/:id/qr`
- [x] `qrcode: false` on instance create — no auto-public QR
- [x] Webhook validates `apikey` header (`EVOLUTION_WEBHOOK_SECRET`)
- [x] Evolution internal URL: `http://sass-botflow_evolution-api:8080` (not public from backend)
- [x] Cloudflare proxy + TLS on public domains
- [x] `DEL_INSTANCE=false` — prevent accidental tenant session deletion
- [x] `LOG_BAILEYS=error` — reduce sensitive log noise
- [ ] Rotate `AUTHENTICATION_API_KEY` quarterly; update backend `EVOLUTION_API_KEY` simultaneously
- [ ] Enable Cloudflare WAF rate limit on `api.botflow.ink/webhooks/evolution`

---

## 7. Scaling Recommendations

### Phase 1 — Single server (current, up to ~50 tenants)

- 1× Evolution API container (2 GB RAM limit)
- 1× Redis 512 MB
- Shared Postgres `evolution` DB
- Polling + webhooks for status

### Phase 2 — 50–200 tenants

- Increase Evolution memory to 4 GB
- Postgres: dedicated `evolution` connection pool (`connection_limit=50`)
- Enable `PROMETHEUS_METRICS=true` + Grafana
- Move message processing to queue (BullMQ via backend `REDIS_URL`)

### Phase 3 — 200+ tenants / HA

- **Horizontal Evolution**: one Evolution instance per N tenants (shard by workspace prefix)
- **Dedicated Postgres** for Evolution (remove shared `sass-botflow_postgres` contention)
- **Redis Cluster** or managed Redis (ElastiCache)
- **Webhook-only** architecture (disable polling in frontend)
- **S3** for media: `S3_ENABLED=true` on Evolution to avoid base64 in webhooks
- Load balancer: route `evolution-{shard}.api.botflow.ink` per shard

### Memory optimization (applied)

| Setting | Value | Why |
|---------|-------|-----|
| `NODE_OPTIONS` | `--max-old-space-size=1536` | Cap Node heap |
| `LOG_BAILEYS` | `error` | Less I/O |
| `DEL_TEMP_INSTANCES` | `true` | Clean orphan instances |
| Redis `maxmemory` | 512mb LRU | Prevent OOM |
| `groupsIgnore` | `true` on create | Skip group sync per tenant |

### PostgreSQL optimization

- `connection_limit=25` in URI (per Evolution container)
- Index maintenance: `REINDEX` monthly on `evolution_api` schema
- Separate DB from main `botflow` app data (already isolated in `evolution` database)

### Redis optimization

- `CACHE_REDIS_SAVE_INSTANCES=true` — fast reconnect after restart
- `CACHE_REDIS_TTL=604800` (7 days)
- `appendonly yes` + `appendfsync everysec` — durability without full sync every write

---

## 8. Feature Matrix

| Requirement | Implementation |
|-------------|----------------|
| Docker | `evoapicloud/evolution-api:v2.3.7` compose |
| Redis | `evolution-redis` service |
| PostgreSQL | `sass-botflow_postgres` / `evolution` DB |
| Webhook | `WEBHOOK_GLOBAL_URL=https://api.botflow.ink/webhooks/evolution` |
| JWT | BotFlow backend JWT on all tenant APIs |
| Multiple instances | `botflow-{workspace}-{suffix}` per tenant |
| One session per user | One `WhatsAppSession` per workspace |
| No public QR | Webhook → Redis → JWT BFF only |
| Reconnect | `CACHE_REDIS_SAVE_INSTANCES=true` |
| Auto session restore | Postgres `DATABASE_SAVE_DATA_INSTANCE` + Redis |
| Message history | `syncFullHistory` + `DATABASE_SAVE_DATA_HISTORIC` |
| Media | Baileys native; enable S3 at scale |
| Typing status | `WEBHOOK_EVENTS_PRESENCE_UPDATE` |
| Read receipts | `readMessages` + `readStatus` on instance |
| Presence | `PRESENCE_UPDATE` webhook |

---

## 9. Deploy Order

1. **Evolution Compose** (this repo `docker-compose.yml`)
2. **Backend** redeploy from `sass-botflow/backend` `main` (includes `/webhooks/evolution`)
3. **Frontend** redeploy from `sass-botflow/frontend` `main`
4. Verify `setup-status` → `ready: true`
5. Test: `/dashboard/whatsapp-profiles` → Add WhatsApp → scan QR

---

## 10. Troubleshooting

| Symptom | Fix |
|---------|-----|
| SSL handshake failure on Evolution | Compose not deployed; redeploy Step 1 |
| Webhook 403 | `EVOLUTION_WEBHOOK_SECRET` ≠ `AUTHENTICATION_API_KEY` |
| QR not showing | Check Redis on backend; check `QRCODE_UPDATED` webhook |
| Session not reconnecting | Verify `CACHE_REDIS_SAVE_INSTANCES=true` + volume `botflow_evolution_instances` |
| `ready: false` backend | Redeploy backend from `main` |
