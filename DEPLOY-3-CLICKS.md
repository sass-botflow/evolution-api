# DEPLOY 3 SERVICES — WhatsApp QR linking

EasyPanel: http://187.124.12.89:3000

---

## SERVICE 1 — evolution-api (Compose)

1. **Source** → `docker-compose.yml` → paste from this repo → replace `YOUR_DB_PASSWORD`
2. **Environment** → DELETE ALL (empty)
3. **Domains** → `evolution.api.botflow.ink` port `8080`
4. **Deploy** (3 min)

Test: https://evolution.api.botflow.ink/health

---

## SERVICE 2 — backend

1. **Source** → GitHub `sass-botflow/backend` branch `main` Dockerfile
2. **Environment** → add:
```
EVOLUTION_API_URL=http://sass-botflow_evolution-api:8080
EVOLUTION_API_KEY=e323292ec09f14a66514089b993e246eb469f3a04448ddc02167447e76621f18
```
3. **Deploy**

Test: `curl -s https://api.botflow.ink/health | jq .modules.whatsapp` → `true`

---

## SERVICE 3 — frontend

1. **Source** → GitHub `sass-botflow/frontend` branch `main`
2. **Environment**:
```
NEXT_PUBLIC_API_URL=https://api.botflow.ink
BACKEND_API_URL=https://api.botflow.ink
JWT_SECRET=<same as backend>
NEXT_PUBLIC_APP_URL=https://www.botflow.ink
```
3. **Deploy** (or GitHub Actions → Deploy now → paste webhook URL)

Test: `curl -s https://www.botflow.ink/api/health | jq .whatsappReady` → `true`

---

## Postgres (done ✅)

```sql
CREATE DATABASE evolution;
```

Password: EasyPanel → postgres → Environment → POSTGRES_PASSWORD

---

## When all green

https://www.botflow.ink/dashboard/whatsapp-profiles → Add WhatsApp → Scan QR
