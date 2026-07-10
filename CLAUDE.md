# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Project Management MVP web app with a Kanban board and AI chat assistant. Users sign in, view/manage a Kanban board with drag-and-drop, and interact with an AI that can create/edit/move cards.

**Stack:** Next.js 16 frontend (React 19, TypeScript, Tailwind v4, @dnd-kit) + Python FastAPI backend + SQLite database, all packaged in Docker. AI via OpenRouter using `openai/gpt-oss-120b`.

**MVP constraints:** Hardcoded login (`user`/`password`), one board per user, local Docker deployment only.

## Commands

### Running the app (Docker)
```bash
# Mac
scripts/start-mac.sh    # Start
scripts/stop-mac.sh     # Stop

# Linux
scripts/start-linux.sh
scripts/stop-linux.sh

# Windows
scripts/start-windows.ps1
scripts/stop-windows.ps1
```
Backend available at http://localhost:8000

### Frontend (from `frontend/` directory)
```bash
npm run dev           # Dev server
npm run build         # Build static export
npm run lint          # ESLint
npm run test:unit     # Vitest unit tests
npm run test:e2e      # Playwright E2E tests
npm run test:all      # All tests
```

### Backend (from `backend/` directory)
```bash
pytest                           # Run all tests
pytest tests/test_main.py        # Single test file
pytest -k "test_name"            # Single test by name
PM_BASE_URL=http://localhost:8000 pytest tests/test_integration.py  # Integration tests against running server
```

### Live integration/E2E testing against a running container
```bash
docker build -t pm-app .
docker rm -f pm-app || true
docker run -d --name pm-app --env-file .env -p 8000:8000 pm-app
curl -sf http://127.0.0.1:8000/health   # wait for readiness
PM_BASE_URL=http://127.0.0.1:8000 PYTHONPATH=$(pwd) pytest backend
npm run test:all --prefix frontend      # Playwright runs against the backend-served app on :8000
```

## Architecture

**Frontend (`frontend/src/`):**
- `app/page.tsx` - Main page with login and Kanban board
- `components/KanbanBoard.tsx` - Board state owner, drag-and-drop handling
- `components/ChatSidebar.tsx` - AI chat interface
- `lib/api.ts` - API client functions
- `lib/kanban.ts` - Data types (Card, Column, BoardData) and utilities

**Backend (`backend/app/`):**
- `main.py` - FastAPI app, lifespan, route registration
- `config.py` - Environment config, constants, seed data
- `models.py` - Pydantic request/response models
- `database.py` - DB connection, init, queries (SQLite tables: users, boards, columns, cards)
- `ai.py` - OpenRouter integration and structured-output action application
- `dependencies.py` - FastAPI dependencies (`get_db`, `get_username`)
- `routes/board.py` - Board/column/card CRUD (`/api/board`, `/api/columns/{id}`, `/api/cards/{id}`)
- `routes/chat.py` - AI chat endpoint (`/api/chat`)
- `routes/static.py` - Serves built frontend static files from `/` in production

**ID Prefixing:** Frontend prefixes IDs with `col-` and `card-` for drag-and-drop stability, strips them for API calls.

**Auth:** No real auth — `get_username` dependency resolves the hardcoded MVP user; all board/column/card queries are scoped to that user's single board.

## Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)
- Dark Navy: `#032147` (headings)
- Gray Text: `#888888`

## Development Guidelines

- Keep it simple. No over-engineering or unnecessary defensive programming.
- Identify root cause before fixing issues. Prove with evidence, then fix.
- Work incrementally with small steps. Validate each increment.
- Use latest library APIs.
- Use `uv` as Python package manager in Docker.
- Planning docs are in `docs/` - review `docs/PLAN.md` for context.


## DETAILED PLAN

@docs/PLAN.md

