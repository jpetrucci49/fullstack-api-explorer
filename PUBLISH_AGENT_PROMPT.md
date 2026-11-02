# Agent prompt: make API Explorer deployable and guide the live publish

Copy everything below the line into an agent whose workspace is this `api-explorer` project.

---

You are working in the API Explorer monorepo. Your job is to (1) make the code actually deployable on free hosts, then (2) walk me through publishing a public live demo, one step at a time, and wait for me to confirm each step before continuing.

## Goal

Publish a public live demo of this project for my resume and LinkedIn, at **$0**. No paid plans. If a host asks for a credit card, stop and tell me the alternative instead of pushing me onto a paid tier.

The live app must let a visitor pick any of the five backends, fetch a GitHub username, see `X-Cache` HIT/MISS, run `/analyze`, and clear the cache.

When we are done, give me one frontend URL I can put on my resume as the Live demo link.

## What this project is

A Next.js frontend that calls the same GitHub profile API implemented five times, with a shared Redis cache:

| Folder | Stack | Local port | GitHub repo |
| --- | --- | --- | --- |
| `frontend/` | Next.js 15 / React 19 | 3000 | `jpetrucci49/api-explorer-frontend` |
| `api-explorer-node/` | Express | 3001 | `jpetrucci49/api-explorer-node` |
| `api-explorer-fastapi/` | FastAPI | 3002 | `jpetrucci49/api-explorer-fastapi` |
| `api-explorer-django/` | Django | 3003 | `jpetrucci49/api-explorer-django` |
| `api-explorer-rails/` | Rails | 3004 | `jpetrucci49/api-explorer-rails` |
| `api-explorer-deno/` | Deno | 3005 | `jpetrucci49/api-explorer-deno` |

Umbrella repo: `jpetrucci49/fullstack-api-explorer` (local folder may be `api-explorer`). Several backends are nested git repos. Edit the files here, then tell me which remotes need a push. Do not force-push. Do not commit secrets.

Endpoints to keep working: `GET /github?username=`, `GET /analyze?username=`, `POST /clear-cache`. Redis keys expire in 1800 seconds. Backends already fall back to live GitHub when Redis is down; keep that.

## Constraints

- I cannot pay for hosting.
- Preferred free stack: **Vercel** (frontend), **Render** free Docker web services (five backends), **Redis Cloud** free 30 MB (not Upstash; these clients do not use TLS).
- If Render requires a credit card, stop and switch the backends to Hugging Face Spaces or another no-card host. Do not keep going as if Render will work.
- Free servers sleep when idle. First request can take 30–60 seconds. Mention that in the README and, if cheap, in the UI.
- Never commit `.env`, `config/master.key`, API keys, or Redis passwords.
- Do not invent GitHub tokens, Redis URLs, or deploy URLs. I will create the accounts and paste values back.
- Keep local `docker-compose` working. Production env vars should default to today’s localhost values when unset.
- Ask one question at a time when you need something from me. After each publish step, wait until I say it worked or paste the error.

## Phase 1 — code changes (do this first, before any hosting)

The current code cannot be published. Fix all of the following.

### 1. Frontend backend URLs

`frontend/src/app/page.tsx` hardcodes:

```ts
{ id: "node", label: "Node.js", url: "http://localhost:3001" },
{ id: "fastapi", label: "FastAPI", url: "http://localhost:3002" },
{ id: "django", label: "Django", url: "http://localhost:3003" },
{ id: "rails", label: "Rails", url: "http://localhost:3004" },
{ id: "deno", label: "Deno", url: "http://localhost:3005" },
```

Replace those with public env vars, with localhost fallbacks for local dev:

- `NEXT_PUBLIC_NODE_API_URL`
- `NEXT_PUBLIC_FASTAPI_API_URL`
- `NEXT_PUBLIC_DJANGO_API_URL`
- `NEXT_PUBLIC_RAILS_API_URL`
- `NEXT_PUBLIC_DENO_API_URL`

Strip trailing slashes. Add `frontend/.env.example` listing those keys with the localhost defaults. Do not commit a real `.env`.

Optional but useful: when the user first loads the page, ping each backend (or show a short “first load may take a minute” note) because Render free instances sleep.

### 2. CORS and hosts (four backends are locked to localhost)

Node already allows any origin via `cors()`. Leave that, or optionally restrict it with `FRONTEND_ORIGIN` the same way as the others.

Make every other backend read `FRONTEND_ORIGIN` (comma-separated origins are fine). Default to `http://localhost:3000` when unset.

Exact places:

- FastAPI `api-explorer-fastapi/main.py`: `common_headers["Access-Control-Allow-Origin"]` and `CORSMiddleware(allow_origins=...)` are both `"http://localhost:3000"`.
- Deno `api-explorer-deno/main.ts`: `commonHeaders["Access-Control-Allow-Origin"]` is `"http://localhost:3000"`. Also handle OPTIONS preflight if needed.
- Django `api-explorer-django/config/settings.py`: `CORS_ALLOWED_ORIGINS = ["http://localhost:3000"]` and `ALLOWED_HOSTS = ["localhost", "0.0.0.0"]`. For production, `ALLOWED_HOSTS` must include the Render hostname (from `RENDER_EXTERNAL_HOSTNAME` or `ALLOWED_HOSTS` env). Set `DEBUG` from env, default True locally, False when `DJANGO_DEBUG=false`.
- Rails `api-explorer-rails/config/initializers/cors.rb`: `origins 'http://localhost:3000'`. Read `FRONTEND_ORIGIN`.

Keep exposing `X-Cache`.

### 3. Bind to `$PORT`

Render and most PaaS inject `PORT`. Dockerfiles currently pin local ports and some use `--reload` / `--watch`, which is wrong in production.

- FastAPI Dockerfile CMD uses `--port 3002 --reload`. Code already has `PORT = int(os.getenv("PORT", 3002))`. Start command must be `uvicorn main:app --host 0.0.0.0 --port $PORT` with no reload.
- Node already uses `process.env.PORT || 3001`. Dockerfile should `CMD ["node", "index.js"]` (not `npm run dev`).
- Django Dockerfile/compose uses gunicorn on 3003. Production: `gunicorn config.wsgi:application --bind 0.0.0.0:$PORT`.
- Rails Dockerfile `CMD ["rails", "server", "-b", "0.0.0.0", "-p", "3004"]`. Production: `-p ${PORT}` or a start script that reads `ENV["PORT"]`.
- Deno already uses `Deno.env.get("PORT") || 3005`. Dockerfile CMD uses `--reload --watch`. Production CMD must drop those: `deno run --allow-net --allow-read --allow-env main.ts`.

Keep compose/dev commands working for local Docker. Prefer a production CMD in the Dockerfile plus a compose `command:` override for dev, or a small start script.

### 4. Redis

Clients use host/port/password **without TLS**. Document that Redis Cloud must have TLS off. Do not switch to Upstash unless you also add TLS in every client.

If Redis is missing, backends must still answer from GitHub (they already try/except this). Do not crash the process on boot if Redis is down.

### 5. README

Update the root README (and backend READMEs if they still say localhost-only):

- One-line pitch
- Live demo URL placeholder
- Note: first load may take a minute (free hosting)
- Local run (`docker-compose up --build`)
- Production env var table
- How to deploy (Vercel + five Render Docker services + Redis Cloud)
- Never commit secrets

### 6. Env examples

Add `.env.example` files where missing. Shared backend vars:

```text
PORT=
GITHUB_TOKEN=
REDIS_HOST=
REDIS_PORT=
REDIS_PASSWORD=
FRONTEND_ORIGIN=http://localhost:3000
```

Django also: `DJANGO_SECRET_KEY`, `DJANGO_DEBUG=true`, `ALLOWED_HOSTS=localhost,0.0.0.0`

Rails also: `RAILS_MASTER_KEY` (from local `config/master.key`; never commit the key)

Frontend: the five `NEXT_PUBLIC_*_API_URL` vars.

When Phase 1 is done, show me a short diff summary and which git remotes to push. Do not start hosting until I confirm the code is pushed.

## Phase 2 — guide me through publishing (one step at a time)

Do not click through my accounts. I will create them. After each step, wait for my URL or error paste.

### Accounts

1. Vercel, GitHub login
2. Render, GitHub login (stop if it demands a card)
3. Redis Cloud free 30 MB, TLS off
4. GitHub fine-grained token, read-only public repos (raises unauthenticated 60 req/hour to 5,000)

### Order of operations

1. Redis Cloud → I paste host, port, password. You confirm TLS is off.
2. GitHub token → I paste that it exists (not the token itself into git).
3. Five Render **Web Services**, Runtime Docker, instance Free, one per backend repo. Give me the exact start-command override and env var list for each:

Suggested start commands:

| Service | Start command |
| --- | --- |
| Node | `node index.js` |
| FastAPI | `uvicorn main:app --host 0.0.0.0 --port $PORT` |
| Django | `gunicorn config.wsgi:application --bind 0.0.0.0:$PORT` |
| Rails | `rails s -b 0.0.0.0 -p $PORT` |
| Deno | `deno run --allow-net --allow-read --allow-env main.ts` |

Same env on all five: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `GITHUB_TOKEN`, `FRONTEND_ORIGIN` (placeholder until Vercel exists), plus Django/Rails extras above.

4. I paste the five `*.onrender.com` URLs. You check `/github?username=octocat` if I can curl them.
5. Vercel: import `api-explorer-frontend` (or umbrella repo with Root Directory `frontend`). Env: the five `NEXT_PUBLIC_*_API_URL` values, no trailing slash.
6. I paste the Vercel URL. Then we set `FRONTEND_ORIGIN` on all five backends to that origin (no trailing slash) and redeploy backends.
7. Test plan you walk me through:
   - Open the Vercel URL in a private window
   - Fetch `octocat` on Node (expect MISS, then HIT)
   - Repeat on FastAPI, Django, Rails, Deno
   - Run analyze
   - Clear cache
   - Confirm CORS is not blocking
8. Put the live URL at the top of the README. Tell me the exact resume string: the Vercel URL as Live demo.

If a backend 502s or CORS-blocks, debug with me using the error text. Do not guess I deployed the wrong repo.

## What not to do

- Do not deploy HealthCore or the Voice Command API. This prompt is only API Explorer.
- Do not require a paid Redis, paid Render, or a custom domain.
- Do not flatten the five backends into one service. The point of the project is five implementations.
- Do not rewrite the app. Only the changes needed to publish.
- Do not commit `config/master.key` or `credentials.yml.enc` secrets. Rails needs `RAILS_MASTER_KEY` set on Render from my local `api-explorer-rails/config/master.key`.
