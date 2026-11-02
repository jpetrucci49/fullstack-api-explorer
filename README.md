# API Explorer

One Next.js page calls the same GitHub profile API implemented five times (Node, FastAPI, Django, Rails, Deno), with a shared Redis cache.

**Live demo:** `https://YOUR-APP.vercel.app` (placeholder until the frontend is deployed)

Free hosts sleep when idle. The first request to a backend can take up to a minute. The page says so.

## What you can do

- `GET /github?username=` returns a GitHub user. The response includes `X-Cache: HIT` or `MISS`.
- `GET /analyze?username=` returns top languages for that user.
- `POST /clear-cache` flushes that backend's Redis database.
- `GET /health` is a liveness check for the host. Rails also serves `GET /up`.

Cached GitHub responses expire after 1800 seconds. If Redis is down, each backend still answers from GitHub.

## Services

| Folder | Stack | Local port | Repo |
| --- | --- | --- | --- |
| `frontend/` | Next.js | 3000 | [api-explorer-frontend](https://github.com/jpetrucci49/api-explorer-frontend) |
| `api-explorer-node/` | Express | 3001 | [api-explorer-node](https://github.com/jpetrucci49/api-explorer-node) |
| `api-explorer-fastapi/` | FastAPI | 3002 | [api-explorer-fastapi](https://github.com/jpetrucci49/api-explorer-fastapi) |
| `api-explorer-django/` | Django | 3003 | [api-explorer-django](https://github.com/jpetrucci49/api-explorer-django) |
| `api-explorer-rails/` | Rails | 3004 | [api-explorer-rails](https://github.com/jpetrucci49/api-explorer-rails) |
| `api-explorer-deno/` | Deno | 3005 | [api-explorer-deno](https://github.com/jpetrucci49/api-explorer-deno) |

The apps live in those six repositories. This repo is the local Compose project and the deploy notes.

## Run locally

```bash
git clone https://github.com/jpetrucci49/fullstack-api-explorer.git
cd fullstack-api-explorer
```

Clone each service repo into the matching folder (or use the copies already in this workspace). Copy each `.env.example` to `.env`. Inside Compose, set `REDIS_HOST=redis` on every backend. Put a GitHub token in `GITHUB_TOKEN` if you have one. The token is optional locally; without it GitHub allows about 60 requests an hour.

```bash
docker-compose up --build
```

- Frontend: `http://localhost:3000`
- Backends: ports 3001–3005
- Redis: the Compose network only. The published default password in `docker-compose.yml` is for local use.

Open the frontend, choose a backend, and fetch `octocat`. Dev servers stay on the Compose `command:` overrides. The Dockerfiles themselves start the production process.

## Production environment

Leave these unset locally and the apps keep today's localhost defaults. Set them on the host. Never commit `.env`, `config/master.key`, tokens, or Redis passwords.

Redis clients speak plain TCP (`redis://`, port 6379). On Redis Cloud, turn TLS off. Do not use a `rediss://` URL.

| Variable | Where | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_NODE_API_URL` | Frontend | Node base URL, no trailing slash |
| `NEXT_PUBLIC_FASTAPI_API_URL` | Frontend | FastAPI base URL |
| `NEXT_PUBLIC_DJANGO_API_URL` | Frontend | Django base URL |
| `NEXT_PUBLIC_RAILS_API_URL` | Frontend | Rails base URL |
| `NEXT_PUBLIC_DENO_API_URL` | Frontend | Deno base URL |
| `PORT` | Each backend | Injected by the host. Local defaults: 3001–3005 |
| `GITHUB_TOKEN` | Each backend | Fine-grained GitHub token, public read only |
| `REDIS_HOST` | Each backend | Redis Cloud host |
| `REDIS_PORT` | Each backend | Redis Cloud port |
| `REDIS_PASSWORD` | Each backend | Redis Cloud password |
| `FRONTEND_ORIGIN` | Each backend | Vercel origin, no trailing slash. Comma-separate if you have more than one |
| `DJANGO_SECRET_KEY` | Django | Any long random string |
| `DJANGO_DEBUG` | Django | `false` on the host. Defaults to debug on |
| `ALLOWED_HOSTS` | Django | Defaults to `localhost,0.0.0.0`. Render's hostname is added from `RENDER_EXTERNAL_HOSTNAME` |
| `RAILS_MASTER_KEY` | Rails | Value of local `config/master.key`. Never commit that file |
| `GITHUB_API_URL` | Django, Rails | Defaults to `https://api.github.com` |

Leave `RAILS_ENV` unset so `rails server` stays in development. Production mode expects a database file that this API does not use.

## Deploy

Host cost target is $0.

1. **Redis Cloud** free 30 MB database. Disable TLS. Copy host, port, and password into each backend's environment.
2. **Render** free Docker web services, one per backend repo. Health check path: `/health`.

   | Service | Start command |
   | --- | --- |
   | Node | `node index.js` |
   | FastAPI | `uvicorn main:app --host 0.0.0.0 --port $PORT` |
   | Django | `gunicorn config.wsgi:application --bind 0.0.0.0:$PORT` |
   | Rails | `rails s -b 0.0.0.0 -p $PORT` |
   | Deno | `deno run --allow-net --allow-read --allow-env main.ts` |

3. **Vercel** for `api-explorer-frontend`. Set the five `NEXT_PUBLIC_*_API_URL` values to the Render URLs, with no trailing slash.
4. Set `FRONTEND_ORIGIN` on all five backends to the Vercel origin (`https://….vercel.app`, no trailing slash) and redeploy the backends.

If Render requires a credit card, stop. Do not pick a paid instance. Use another no-card host for the five Docker services and keep this same env layout.

## Secrets

`.env` files, `api-explorer-rails/config/master.key`, and real tokens stay on your machine and in the host's secret fields. `.env.example` files list the keys only.

---

Built by Joseph Petrucci
