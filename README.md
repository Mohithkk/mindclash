# MindClash MVP

A full-stack multiplayer problem-solving game with registration/login, four quiz arenas, rating-aware 1v1 matchmaking, live match updates, first-correct-answer scoring, Elo updates, a leaderboard, and an optional practice bot.

## Stack
- Frontend: React + Vite
- Backend: FastAPI + SQLAlchemy
- Database: SQLite for local development (set `DATABASE_URL` to use PostgreSQL)
- Matchmaking: Redis sorted sets + an atomic Lua pairing script
- Live match updates: authenticated WebSockets (HTTP polling fallback in the browser)
- Practice opponent: built-in rating-adjusted bot; no external AI API key required

## Run locally

### 1. Start Redis

From the project root:

```bash
docker compose up -d redis
```

Check it:

```bash
docker compose exec redis redis-cli ping
# PONG
```

If Redis is already installed locally, you can start it with `redis-server` instead.

### 2. Start the backend

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate       # Windows: .venv\\Scripts\\activate
pip install -r requirements.txt
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

The backend expects Redis at `redis://localhost:6379/0` by default. Override with `REDIS_URL` if needed. Set a strong `JWT_SECRET` before deployment.

### 3. Start the frontend (new terminal)

```bash
cd frontend
npm install
npm run dev
```

Open http://localhost:5173. API docs are at http://localhost:8000/docs.

## Test human matchmaking

1. Register Player A in a normal browser window.
2. Register Player B in an incognito/private window (or another browser).
3. Both browser windows must use this same backend and database.
4. Choose the same arena and click **Find opponent** in both windows.
5. The Redis queue pairs the closest eligible ratings (initial range ±100; it widens as players wait).
6. Both clients receive match updates over WebSockets. If a WebSocket disconnects, the frontend falls back to HTTP polling.

You can also click **Play practice bot instead** from the matchmaking screen. The built-in practice opponent uses a rating-adjusted solve probability and runs locally without an external model.

## Matchmaking and concurrency notes

- Redis sorted sets hold per-arena waiting queues.
- A Lua script selects the closest eligible rating and removes both users in one atomic Redis operation, preventing two requests from claiming the same queued player.
- Redis queue entries expire after 10 minutes of inactivity; users can cancel search.
- The match answer endpoint uses a conditional database update on `(match_id, current_question, status)` so only one correct submission can claim and advance a question.
- WebSocket connections are managed in the FastAPI process. For multi-worker/multi-instance production deployments, add Redis Pub/Sub (or a dedicated event broker) for cross-process WebSocket fan-out, and use PostgreSQL row locking/transactions.
- The built-in bot is a heuristic practice opponent, not a generative AI/LLM. It is intentionally simple and should be labelled as a practice bot in the product UI.

## Current MVP limitations

- The question bank contains demo multiple-choice questions, not a curated production bank.
- The DSA arena currently tests concepts rather than executing arbitrary source code. Do not execute user-submitted code inside the API process; add a separate isolated execution service if code challenges are introduced.
- The bot runs as a background task in the API process. In production, move bot jobs to a durable worker/queue.
- Add email verification, password reset, rate limiting, abuse controls, observability, backups, and stronger deployment security before public launch.
