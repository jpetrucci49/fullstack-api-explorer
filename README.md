# API Explorer

A multi-backend API exploration project with a React/Next.js frontend, querying GitHub user data via five distinct backend implementations. Built to compare server-side technologies while leveraging a shared Redis cache.

## Overview

This repository contains six sub-projects:
- **Frontend**: React/Next.js/TypeScript/Tailwind CSS client ([frontend/](frontend/)).
- **Backends**: Five RESTful APIs querying GitHub:
  - [Deno](api-explorer-deno/) (port 3005)
  - [Django](api-explorer-django/) (port 3003)
  - [FastAPI](api-explorer-fastapi/) (port 3002)
  - [Node.js/Express](api-explorer-node/) (port 3001)
  - [Rails](api-explorer-rails/) (port 3004)

## Features

- **Shared Redis Cache**: All backends cache GitHub API responses (`github:jpetrucci49`) for 30 minutes.
- **Endpoint**: `/github?username={username}` returns GitHub user details (e.g., login, name, repos).
- **Frontend**: Select a backend, input a username, and view cached or live data with `X-Cache` status.

## Setup

1. **Clone the repo**  
   ```bash
   git clone https://github.com/jpetrucci49/api-explorer.git
   cd api-explorer
   ```
2. **Run with Docker**  
   ```bash
   docker-compose up --build
   ```
   - Frontend: `http://localhost:3000`
   - Backends: `http://localhost:3001-3005`
   - Redis: Internal (port 6379)
3. **Manual Setup**: See individual READMEs for local runs without Docker.

## Usage

- Start all services with `docker-compose up`.
- Open `http://localhost:3000`, pick a backend, enter "octocat", and fetch data.
- Check `X-Cache: HIT` or `MISS` in browser dev tools (Network tab).

## Next Steps

- Add `/analyze` endpoint for GitHub profile insights (e.g., top languages).
- Add `/network` endpoint for collaboration graph mapping.
- Enhance frontend with data visualizations (e.g., charts, graphs).
- Deploy to a cloud platform (e.g., Vercel for frontend, Render for backends).

---
Built by Joseph Petrucci | March 2025