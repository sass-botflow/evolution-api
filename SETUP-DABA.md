# FIX DABA — WhatsApp link (5 minutes)

## A. evolution-api → docker-compose.yml

1. Open **Source** → tab **docker-compose.yml**
2. **Delete everything** → paste file from GitHub `docker-compose.yml`
3. Replace **only 2 lines**:
   - `YOUR_API_KEY` → output of `openssl rand -hex 32`
   - `YOUR_DB_PASSWORD` → postgres password for user `botflow`
4. **Save**

## B. evolution-api → Environment

**DELETE ALL** — tab must be **empty**. Save.

## C. evolution-api → Deploy

Click **Deploy** → wait **3 minutes**.

Logs must show:
```
evolution-redis | Ready to accept connections
evolution-api   | Server listening on port 8080
```

## D. backend → Environment

Add or update:
```env
EVOLUTION_API_URL=http://sass-botflow_evolution-api:8080
EVOLUTION_API_KEY=<same YOUR_API_KEY from step A>
```

**Redeploy backend.**

## E. Test

Browser: `https://evolution.api.botflow.ink/health` → must show JSON

App: `https://www.botflow.ink/dashboard/whatsapp-profiles` → **Add WhatsApp** → QR

---

## If deploy fails

| Error | Fix |
|-------|-----|
| `invalid interpolation` | No `${}` in compose — use YOUR_API_KEY literals |
| `network easypanel-sass-botflow not found` | Change network name to `sass-botflow_default` and retry |
| `password authentication failed` | Wrong YOUR_DB_PASSWORD |
| `database evolution does not exist` | `CREATE DATABASE evolution;` in postgres bash |
| `Service is not reachable` | Containers not running — check Logs |

## If network error

In docker-compose.yml change:
```yaml
networks:
  sass-botflow:
    external: true
    name: sass-botflow_default
```
Or try `easypanel-sass-botflow` — depends on EasyPanel version.

## Delete old broken service

Delete **api-botflow-wtsp** if still in sidebar.
