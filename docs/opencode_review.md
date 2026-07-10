# Code Review -- Full Repository (opencode)

Date: 2026-06-24
Scope: entire repository (backend, frontend, Docker, scripts), reviewed against the MVP constraints in `AGENTS.md` and `docs/PLAN.md` ("keep it simple, no over-engineering"). Items that are reasonable simplifications for a local-only, single-user MVP (hardcoded credentials, no rate limiting, single SQLite file, no multi-worker concerns) are intentionally excluded unless they create a real correctness/security risk.

## Priority summary

| # | Severity | Issue | Location |
|---|----------|-------|----------|
| 1 | **Critical** | Path traversal / arbitrary file read in static fallback route | `backend/app/routes/static.py:43-60` |
| 2 | **Critical** | Client-controlled `X-User` header bypasses the "one user" model -- unauthenticated impersonation | `backend/app/dependencies.py:18-19` |
| 3 | **High** | Column rename fires one PATCH request per keystroke (no debounce) | `frontend/src/app/page.tsx:94-107`, `KanbanColumn.tsx:48-53` |
| 4 | **High** | `ValidationError` from AI structured output surfaces as raw 500 (uncaught) | `backend/app/ai.py:41` |
| 5 | **High** | No tests for board-fetch / chat error UI paths or card mutation rollback branches | `frontend/src/app/page.tsx` |
| 6 | **Medium** | Silent no-ops on invalid AI action references with no logging | `backend/app/ai.py:128-260` |
| 7 | **Medium** | Position-clamping logic duplicated 4x across files with already-drifted styles | `backend/app/ai.py`, `backend/app/routes/board.py` |
| 8 | **Medium** | Duplicate `get_db` dependency defined twice (dead code) | `backend/app/database.py:78-83` |
| 9 | **Medium** | Cross-column drag-and-drop targeting is fragile, undocumented, and untested | `frontend/src/components/KanbanBoard.tsx:84-130` |
| 10 | **Medium** | No request sequencing -- overlapping `refreshBoard()` calls can apply stale data | `frontend/src/app/page.tsx` |
| 11 | **Medium** | `@dnd-kit/sortable` v10.0.0 installed alongside `@dnd-kit/core` v6.3.1 (major-version mismatch) | `frontend/package.json` |
| 12 | **Low** | No length cap on chat `message`/`history` sent to paid LLM API | `backend/app/models.py:30-33,73-76` |
| 13 | **Low** | API response/status-code inconsistency (200 vs 201/204, varying envelopes) | `backend/app/routes/board.py` |
| 14 | **Low** | Missing `onDragCancel` handler leaves stale drag state | `frontend/src/components/KanbanBoard.tsx` |
| 15 | **Low** | `handleDeleteCard` silently no-ops on NaN instead of setting `boardError` | `frontend/src/app/page.tsx:132-136` |
| 16 | **Low** | All column-title inputs share identical `aria-label="Column title"` | `frontend/src/components/KanbanColumn.tsx:54` |
| 17 | **Low** | Duplicated background-gradient decoration in 3 places | `frontend/src/app/page.tsx`, `KanbanBoard.tsx` |
| 18 | **Low** | Test coverage gaps: AI action validation boundaries, structured-output failure paths, ID-prefix helper unit tests, drag simulation unit tests | backend + frontend tests |
| 19 | **Low** | Stale `scripts/AGENTS.md` reads "This folder will contain start and stop scripts" | `scripts/AGENTS.md` |
| 20 | **Low** | Start scripts don't pre-check `.env` existence before `--env-file` | `scripts/start-*.sh`, `start-windows.ps1` |

---

## Backend

### Critical

**Path traversal / arbitrary file read in static fallback** -- `backend/app/routes/static.py:43-60`

`requested_path = STATIC_DIR / full_path` joins the path parameter with no normalization or containment check. A request like `GET /../../etc/passwd` resolves outside `STATIC_DIR` and is served via `FileResponse`. The route is unauthenticated, making this trivially exploitable on any running instance.

Fix: resolve and verify containment before serving:
```python
requested_path = (STATIC_DIR / full_path).resolve()
if not requested_path.is_relative_to(STATIC_DIR.resolve()):
    return HTMLResponse("Not found", status_code=404)
```

**Client-controlled `X-User` header bypasses auth** -- `backend/app/dependencies.py:18-19`

```python
def get_username(x_user: str | None = Header(default=None)) -> str:
    return x_user or DEFAULT_USER
```

There is no session/credential enforcement on the API. Identity comes purely from a client-supplied header that the frontend sets on every request. `get_or_create_user` silently creates a new user/board for any string sent. This is not a reasonable MVP simplification -- it is a free multi-tenancy bypass that makes the frontend login screen purely cosmetic at the API layer.

Fix: ignore client-supplied identity and hardcode to `DEFAULT_USER` until real auth exists, removing the unused attack surface:
```python
def get_username() -> str:
    return DEFAULT_USER
```

### High

**Uncaught `ValidationError` in AI response parsing** -- `backend/app/ai.py:41`, called from `backend/app/routes/chat.py:23`

`StructuredChatOutput.model_validate(data)` can raise `pydantic.ValidationError` (e.g. model omits `reply` or returns an unrecognized action `type`). This propagates as a generic 500 instead of the clean `502` used for the sibling JSON-decode failure just above it.

Fix: wrap in `try/except ValidationError` and raise `HTTPException(status_code=502, detail="AI response failed validation")`.

### Medium

- **Silent no-ops on invalid AI action references** -- `backend/app/ai.py:128-260`. Every action type does a bare `continue` when a referenced card/column is not found, with no logging. The chat UI shows a "successful" reply while the mutation silently failed, and there is no server-side way to diagnose this. Add `logger.warning(...)` at minimum; consider surfacing a `warnings` field in `ChatResponse`.

- **Position-clamping logic duplicated 4x** -- `max(0, min(...))` appears in `ai.py` (lines 149-150, 193-194) and `board.py` (lines 49-50, 136-137) with styles already diverging. Extract a shared `clamp_position(position, max_value)` helper in `database.py`.

- **Duplicate `get_db` dependency** -- defined identically in both `backend/app/database.py:78-83` and `backend/app/dependencies.py:10-15`. The `database.py` copy is dead code (routes import from `dependencies`). Delete one copy.

- **`fetch_board` string/int type mismatch** -- `database.py:120-133` builds `cards_by_id` keyed by `str(row["id"])` but `cards_by_column` groups by `int(row["id"])`. Works because Python dict keys are heterogeneous but fragile; inconsistent if a caller expects all keys to be the same type.

### Low

- `ChatRequest.message` / `ChatHistoryItem.content` have no `max_length`, unlike card title/details fields (which have `max_length=500`/`max_length=5000`). Since this is forwarded to a paid OpenRouter call, add a reasonable cap (e.g. 4000 chars) and a capped history length.
- Inconsistent REST conventions: creates return `200` with `{"id": ...}` instead of `201`; updates/deletes return `{"status": "ok"}` instead of `204` or the updated resource. Minor for MVP.
- `apply_actions` is a single ~130-line branchy function; splitting into `_apply_create_card`/`_apply_update_card`/etc. would make each action independently unit-testable.
- Response handlers are typed as bare `dict` rather than Pydantic response models, so `/docs` doesn't document `/api/board` and column/card route shapes.

### Test coverage gaps (backend)

- No test for `apply_actions` with dangling `cardId`/`columnId` references (the most likely real-world AI failure mode).
- No test for `parse_structured_output`'s brace-extraction fallback or the "completely invalid" 502 path -- only the happy path is covered.
- No test for a `StructuredChatOutput` validation failure (missing discriminator, invalid action type) -- currently would 500 uncaught per High finding above.
- No test for `call_openrouter` receiving a non-JSON 200 response.
- No test for Pydantic validation boundaries on `CardCreate`/`CardUpdate`/`ColumnCreate`/`ColumnUpdate` (empty/over-length title, negative position via public API).
- No test for `update_card`/`update_column` 404 when the referenced column exists but belongs to a different board.
- No test for the static path-traversal bug above -- this is why it went unnoticed.

---

## Frontend

### High

**Column rename has no debounce** -- `frontend/src/app/page.tsx:94-107` (`handleRenameColumn`), wired to `KanbanColumn.tsx:48-53`'s `<input onChange>`. Every keystroke fires an immediate `await updateColumn(...)` PATCH with no debounce or ordering guarantee -- a fast typist generates one request per character, and out-of-order responses could revert a title mid-typing.

Fix: debounce (~300-400ms) or only persist on blur, updating local state immediately for responsiveness.

**No tests for board-fetch / chat error UI paths** -- `boardError` (set in `refreshBoard`'s catch, `page.tsx:43-46`) and `chatError` (`page.tsx:205-211`) are never triggered or asserted anywhere. The entire error-banner UX and assistant fallback message are unverified.

**No tests for add/delete/move-card failure branches** -- the optimistic-update-then-rollback catch blocks (`page.tsx:125-129,140-144,171-175`) have 0% test coverage despite being a meaningfully different code path from the happy path.

### Medium

- **Fragile cross-column drag resolution** -- `KanbanBoard.tsx:84-130` mixes three signals (`overIdFromData`, `overColumnFromData`, `lastOverId.current`) with implicit precedence; dropping a card on top of another card in a different column always appends to the end. May be intentional but is undocumented and untested beyond "lands somewhere in the column" (`tests/kanban.spec.ts:31-54`).

- **`@dnd-kit/sortable` v10.0.0 with `@dnd-kit/core` v6.3.1** -- The installed versions show a major-version mismatch (`sortable` is 10.x, `core` is 6.x). While basic usage works, `useSortable` and `SortableContext` in v10 have different internal APIs. Edge cases around collision detection and keyboard sorting may behave differently. Lock compatible versions.

- **`initialData` uses a different ID scheme from API data** -- `kanban.ts:28-82` uses `col-backlog`, `card-1` while the backend returns numeric IDs. `toBoardData()` in `api.ts` prefixes `"col-"`/`"card-"` to backend IDs. `initialData` is dead code used only in `KanbanBoard.test.tsx`, meaning tests don't exercise the real API-driven data flow.

- **No request sequencing for `refreshBoard()`** -- `handleAddCard`/`handleDeleteCard`/`handleMoveCard` each call `refreshBoard()` independently with no de-duplication or cancellation; rapid interactions can cause overlapping fetches to land out of order.

- **No `onDragCancel` handler on `DndContext`** (`KanbanBoard.tsx:277-284`) -- a cancelled drag (e.g. Escape key) can leave stale `activeCardId`/`lastOverId` for the next drag.

- **`handleDeleteCard` silently no-ops on NaN** (`page.tsx:132-136`) -- unlike sibling handlers that set `boardError` on invalid IDs, `handleDeleteCard` returns silently when `Number(fromCardId(cardId))` is NaN. Inconsistent error UX.

- **Column-title inputs share identical `aria-label="Column title"`** (`KanbanColumn.tsx:54`) -- screen readers cannot distinguish between columns. Each should include a unique identifier.

- **Duplicated background-gradient markup** repeated verbatim in 3 places (`page.tsx:220-221,293`, `KanbanBoard.tsx:186-187`) -- a shared `BackgroundDecor` component would remove the duplication.

### Low

- Drag listeners are spread onto the entire card `<article>` including the Remove button area (`KanbanCard.tsx:33-34`); works via the `distance: 6` activation constraint but is a fragile coupling.
- Inline arrow functions recreated every render through 3+ component levels -- no real perf impact at MVP scale, not worth fixing now.

### Security

No XSS issues found -- no `dangerouslySetInnerHTML` anywhere; all user content goes through JSX text interpolation (React auto-escapes). `.env` is correctly gitignored. The `X-User` header trust is a backend concern (see Critical finding #2), not a frontend defect.

### Test coverage gaps (frontend)

- `KanbanBoard.test.tsx` never exercises actual dnd-kit drag simulation (`handleDragStart`/`handleDragOver`/`handleDragEnd`/collision detection) -- only Playwright covers one simple cross-column move.
- Chat-driven board update is tested only for the success case -- no test for actions-without-board, in-flight `isSending` state, or multi-turn history threading.
- `kanban.test.ts` covers `moveCard` but has zero direct tests for the `toCardId`/`fromCardId`/`toColumnId`/`fromColumnId` ID-prefixing round-trip helpers, despite these being load-bearing for every API call.

---

## Docker / Scripts

- Multi-stage Dockerfile is correct: builds the frontend static export, copies only `out/` into the final Python image, uses `uv` per project convention, drops to a non-root user, and includes a `HEALTHCHECK`. No issues found.
- `scripts/start-*.sh` / `start-windows.ps1` don't pre-check that `.env` exists before passing `--env-file` to `docker run` -- failure mode is a raw Docker error rather than a message pointing at `OPENROUTER_API_KEY`. Minor UX polish.
- Stop scripts are minimal, correct, and appropriately simple for a single named container.
- `scripts/AGENTS.md` is stale: reads "This folder will contain start and stop scripts" -- the scripts already exist and have been working.

---

## Strengths

- Clean, idiomatic FastAPI structure with separated concerns (config, database, models, routes, AI).
- Good use of Pydantic for request/response validation with detailed schema definitions.
- Frontend properly uses `@dnd-kit` with typed collision detection strategy and `MeasuringStrategy.Always`.
- No XSS issues -- all user content is safely handled through React's text interpolation.
- Environment secrets (`.env`) correctly gitignored at both root and frontend levels.
- Color scheme consistently applied via CSS variables in `globals.css`.
- Multi-stage Docker build is efficient: frontend built separately, only static output copied into final Python image.
- Effective use of non-root user (`appuser`) in container.
- The `resequence_positions` helper in `database.py` is a good abstraction for maintaining stable ordering.
- Frontend `apiFetch` wrapper provides consistent timeout, error handling, and auth header injection.
- Good use of `useCallback` and `useMemo` in frontend components to prevent unnecessary re-renders.
- Comprehensive E2E test with Playwright covering login, card creation, and cross-column drag-and-drop.
- Backend test for missing API key returns clean 500 rather than crashing.
- Mock-based structured chat test validates the full round-trip from API call to board update.

---

## Recommended next steps (in order)

1. Fix the static-file path traversal (`backend/app/routes/static.py`) -- exploitable today, highest impact, smallest fix.
2. Lock down `X-User` to `DEFAULT_USER` only, matching the stated "one board per user" MVP design.
3. Debounce column rename on the frontend (or persist only on blur).
4. Wrap `StructuredChatOutput.model_validate()` in `try/except ValidationError` for a clean error path.
5. Add backend tests for AI action dangling-reference handling and the uncaught `ValidationError` path.
6. Add frontend tests for the board/chat error states and the card mutation rollback branches -- currently the largest blind spot in frontend coverage.
7. Address the Medium/Low items opportunistically as related code is touched (duplicate `get_db`, position-clamp duplication, drag-and-drop targeting documentation).
