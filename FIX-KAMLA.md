# FIX KAMLA — 10 dqiqa

## PROBLEM 1: Evolution ma runningch
## PROBLEM 2: Gateway Error f WhatsApp page

---

# PART A — EVOLUTION (5 min)

## A1. docker-compose.yml
EasyPanel → evolution-api → Source → docker-compose.yml

**Delete kolchi** → paste from GitHub `docker-compose.yml`

**Bdel ghir wa7da:** `YOUR_DB_PASSWORD` → password dyal user `botflow`

Password kaynin f: EasyPanel → **postgres** → **Environment** → `POSTGRES_PASSWORD`

## A2. Environment
**DELETE KOLCHI** — khalli **khawi** → Save

## A3. Deploy
Deploy → stenna 3 dqiqa

## A4. Test
https://evolution.api.botflow.ink/health → JSON

---

# PART B — BACKEND (2 min)

EasyPanel → **backend** → Environment → zid:

```env
EVOLUTION_API_URL=http://sass-botflow_evolution-api:8080
EVOLUTION_API_KEY=e323292ec09f14a66514089b993e246eb469f3a04448ddc02167447e76621f18
```

**Redeploy backend**

---

# PART C — FRONTEND Gateway Error (3 min)

EasyPanel → **frontend** → Environment → verify:

```env
NEXT_PUBLIC_API_URL=https://api.botflow.ink
BACKEND_API_URL=https://api.botflow.ink
JWT_SECRET=<COPY MN BACKEND — nfs value>
NEXT_PUBLIC_APP_URL=https://www.botflow.ink
```

⚠️ `JWT_SECRET` frontend = `JWT_SECRET` backend (**character b character**)

EasyPanel → frontend → Source → verify GitHub `sass-botflow/frontend` branch `main`

**Redeploy frontend**

Test: https://www.botflow.ink/api/health
→ `"reachable": true`

---

# PART D — LINK WHATSAPP

1. https://www.botflow.ink/dashboard/whatsapp-profiles
2. Add WhatsApp
3. Scan QR

---

# Ila Compose ma khdamch — PLAN B (App mode)

1. Delete evolution-api compose service
2. Add Service → **App** (machi Compose)
3. Repo: sass-botflow/evolution-api, branch main
4. Dockerfile: `Dockerfile`
5. Port: 8080
6. Environment (paste all):

```env
SERVER_URL=https://evolution.api.botflow.ink
SERVER_PORT=8080
AUTHENTICATION_API_KEY=e323292ec09f14a66514089b993e246eb469f3a04448ddc02167447e76621f18
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://botflow:YOUR_DB_PASSWORD@187.124.12.89:5432/evolution?schema=evolution_api
DATABASE_CONNECTION_CLIENT_NAME=botflow_evolution
CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=redis://sass-botflow_redis:6379/1
CACHE_REDIS_PREFIX_KEY=botflow_evolution
WEBSOCKET_ENABLED=false
WEBHOOK_GLOBAL_ENABLED=false
LOG_LEVEL=ERROR,WARN,INFO
LANGUAGE=en
```

7. Domain: evolution.api.botflow.ink → 8080
8. Deploy

---

# DELETE
- **api-botflow-wtsp** (service qdim ghalat)
