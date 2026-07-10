# Code Review — Full Repository

Date: 2026-06-22
Scope: entire repository (backend, frontend, Docker, scripts), reviewed against the MVP constraints in `CLAUDE.md` and `docs/PLAN.md` ("keep it simple, no over-engineering"). Items that are reasonable simplifications for a local-only, single-user MVP (hardcoded credentials, no rate limiting, single SQLite file, no multi-worker concerns) are intentionally excluded unless they create a real correctness/security risk.

## Priority summary

| # | Severity | Issue | Location |
|---|----------|-------|----------|
| 1 | **Critical** | Path traversal / arbitrary file read in static file fallback route | `backend/app/routes/static.py:42-60` |
| 2 | **High** | Client-controlled `X-User` header bypasses the "one user" model — unauthenticated impersonation | `backend/app/dependencies.py:18-19` |
| 3 | **High** | Column rename fires one PATCH request per keystroke (no debounce) | `frontend/src/app/page.tsx:94-107`, `KanbanColumn.tsx:48-53` |
| 4 | **High** | No tests for board-fetch / chat error UI paths | `frontend/src/app/page.tsx` (`boardError`, `chatError`) |
| 5 | **High** | No tests for add/delete/move-card failure (rollback) branches | `frontend/src/app/page.tsx:125-176` |
| 6 | Medium | Uncaught `ValidationError` from AI structured output surfaces as raw 500 | `backend/app/ai.py:41` |
| 7 | Medium | `update_card` writes title/details before validating target column exists | `backend/app/routes/board.py:190-210` |
| 8 | Medium | `apply_actions` silently no-ops on invalid AI-supplied references, no logging | `backend/app/ai.py:128-260` |
| 9 | Medium | Duplicate `get_db` dependency defined twice (dead code) | `backend/app/database.py:78-83` |
| 10 | Medium | Drag-and-drop cross-column drop targeting is fragile/undocumented and untested | `frontend/src/components/KanbanBoard.tsx:84-130` |
| 11 | Medium | No request sequencing — overlapping `refreshBoard()` calls can apply stale data | `frontend/src/app/page.tsx` |
| 12 | Low | Duplicated position-clamping logic (4 copies) | `backend/app/ai.py`, `backend/app/routes/board.py` |
| 13 | Low | No length cap on chat `message`/`history` sent to paid LLM API | `backend/app/models.py:30-33,73-76` |
| 14 | Low | API response/status-code inconsistency (200 vs 201/204, varying envelopes) | `backend/app/routes/board.py` |
| 15 | Low | Missing `onDragCancel` handler leaves stale drag state | `frontend/src/components/KanbanBoard.tsx:238-245` |
| 16 | Low | Misc test gaps: AI action validation boundaries, structured-output failure paths, ID-prefix helper unit tests, drag-and-drop unit tests | backend + frontend tests |
| 17 | Low | Minor polish: aria-label disambiguation, duplicated gradient markup, `.env` pre-flight check in start scripts | various |

---

## Backend

### Critical

**Path traversal / arbitrary file read in static fallback** — `backend/app/routes/static.py:42-60`

`requested_path = STATIC_DIR / full_path` joins the path parameter directly with no normalization or containment check, so a request like `GET /../../tmp/secret.txt` resolves outside `STATIC_DIR` and is served via `FileResponse`. Verified live — this returns arbitrary readable files from the container (e.g. anything readable by `appuser`). The route is unauthenticated, so this is a real, trivially exploitable bug, not an MVP simplification.

Fix: resolve and verify containment before serving:
```python
requested_path = (STATIC_DIR / full_path).resolve()
if not requested_path.is_relative_to(STATIC_DIR.resolve()):
    return HTMLResponse("Not found", status_code=404)
```

### High

**Client-controlled `X-User` header bypasses auth** — `backend/app/dependencies.py:18-19`

```python
def get_username(x_user: str | None = Header(default=None)) -> str:
    return x_user or DEFAULT_USER
```
There is no session/credential enforcement on the API at all; identity comes purely from a client-supplied header. `get_or_create_user` will silently create a new user/board for any string sent. This exceeds "reasonable MVP simplification" because it's a free multi-tenancy bypass that the frontend login screen does nothing to prevent at the API layer.

Fix: ignore client-supplied identity and hardcode to `DEFAULT_USER` until real auth exists, removing the unused attack surface.

### Medium

- **Uncaught `ValidationError` in AI response parsing** — `backend/app/ai.py:41`, called from `backend/app/routes/chat.py:23`. `StructuredChatOutput.model_validate(data)` can raise `pydantic.ValidationError` (e.g. model omits `reply` or returns an unrecognized action `type`). This propagates as a generic 500 instead of the clean `502` used for the sibling JSON-decode failure just above it. Wrap it in a `try/except ValidationError` and raise `HTTPException(502, ...)`.

- **`update_card` partial-write ordering** — `backend/app/routes/board.py:190-210`. Title/details are updated before validating that `payload.column_id` belongs to the board; a 404 on bad `column_id` happens after the write already executed (relies on implicit no-commit-no-op behavior on close). Validate `column_id` before issuing any `UPDATE`.

- **Silent no-ops on invalid AI action references** — `backend/app/ai.py:128-260`. Every action type does a bare `continue` when a referenced card/column isn't found, with no logging. The chat UI will show a "successful" reply while the mutation silently failed, and there is no way to diagnose this server-side. Add `logger.warning(...)` at minimum; consider surfacing a `warnings` field in `ChatResponse`.

- **Duplicate `get_db` dependency** — defined identically in both `backend/app/database.py:78-83` and `backend/app/dependencies.py:10-15`. The `database.py` copy is dead code (routes import from `dependencies`). Delete one copy.

### Low

- Position-clamping logic (`max(0, min(...))`) duplicated 4× across `ai.py` and `board.py` with inconsistent styles already drifting. Extract a shared `clamp_position()` helper in `database.py`.
- `ChatRequest.message` / `ChatHistoryItem.content` have no `max_length`, unlike card title/details fields. Since this is forwarded to a paid OpenRouter call, add a reasonable cap (e.g. 4000 chars) and a capped history length.
- Inconsistent REST conventions: creates return `200` with `{"id": ...}` instead of `201`; updates/deletes return `{"status": "ok"}` instead of `204` or the updated resource. Not wrong, just non-idiomatic — fine to leave for MVP.
- `apply_actions` is a single ~130-line branchy function; splitting into `_apply_create_card`/`_apply_update_card`/etc. would make each action independently unit-testable.
- Response/handler functions are typed as bare `dict` rather than Pydantic response models, so `/docs` doesn't document `/api/board` and column/card route shapes.

### Test coverage gaps (backend)

- No test for `apply_actions` with dangling `cardId`/`columnId` references (the most likely real-world AI failure mode).
- No test for `parse_structured_output`'s brace-extraction fallback or the "completely invalid" 502 path — only the happy path is covered.
- No test for a `StructuredChatOutput` validation failure (missing discriminator etc.) — currently would 500 uncaught, see Medium finding above.
- No test for `call_openrouter` receiving a non-JSON 200 response.
- No test for Pydantic validation boundaries on `CardCreate`/`CardUpdate`/`ColumnCreate`/`ColumnUpdate` (empty/over-length title, negative position via public API).
- No test for `update_card`/`update_column` 404 when the referenced column exists but belongs to a different board.
- No test for the static path-traversal bug above — this is why it went unnoticed.

---

## Frontend

### High

**Column rename has no debounce** — `frontend/src/app/page.tsx:94-107` (`handleRenameColumn`), wired to `KanbanColumn.tsx:48-53`'s `<input onChange>`. Every keystroke fires an immediate `await updateColumn(...)` PATCH with no debounce or ordering guarantee — a fast typist generates one request per character, and out-of-order responses could revert a title mid-typing. Fix: debounce (~300-400ms) or only persist on blur, updating local state immediately for responsiveness.

**No tests for board-fetch / chat error UI paths** — `boardError` (set in `refreshBoard`'s catch, `page.tsx:43-46`) and `chatError` (`page.tsx:205-211`) are never triggered or asserted anywhere. The entire error-banner UX and assistant fallback message are unverified.

**No tests for add/delete/move-card failure branches** — the optimistic-update-then-rollback catch blocks (`page.tsx:125-129,140-144,171-175`) have 0% test coverage despite being a meaningfully different code path from the happy path.

### Medium

- **Fragile cross-column drag resolution** — `KanbanBoard.tsx:84-130` mixes three signals (`overIdFromData`, `overColumnFromData`, `lastOverId.current`) with implicit precedence; dropping a card directly on top of another card in a different column always appends to the end rather than inserting at the hovered position. This may be intentional but is undocumented and untested beyond "lands somewhere in the column" (`tests/kanban.spec.ts:31-54`).
- **No request sequencing for `refreshBoard()`** — `handleAddCard`/`handleDeleteCard`/`handleMoveCard` each call `refreshBoard()` independently with no de-duplication or cancellation; rapid interactions can cause overlapping fetches to land out of order. Track a sequence number/AbortController, or accept as low-risk for single-user MVP usage.
- Duplicated background-gradient decoration markup repeated verbatim in 3 places (`page.tsx:220-221,293`, `KanbanBoard.tsx:186-187`) — small `BackgroundDecor` component would remove the duplication.
- Inline arrow functions recreated every render through 3+ component levels — no real perf impact at MVP scale, not worth fixing now.

### Low

- No `onDragCancel` handler on `DndContext` (`KanbanBoard.tsx:238-245`) — a cancelled drag (e.g. `Escape`) can leave stale `activeCardId`/`lastOverId` for the next drag.
- `handleDeleteCard` silently no-ops on `NaN` card id instead of setting `boardError` like its sibling handlers (`page.tsx:132-136`) — inconsistent error UX, low real-world impact.
- Column-title inputs all share the identical `aria-label="Column title"` with no per-column disambiguation — screen readers can't distinguish between columns.
- Drag listeners are spread onto the entire card `<article>` including the Remove button area (`KanbanCard.tsx:33-34`); works via the `distance: 6` activation constraint but is a fragile coupling.

### Test coverage gaps (frontend)

- `KanbanBoard.test.tsx` never exercises actual dnd-kit drag simulation (`handleDragStart`/`handleDragOver`/`handleDragEnd`/collision detection) — only Playwright covers one simple cross-column move.
- Chat-driven board update is tested only for the success case — no test for actions-without-board, in-flight `isSending` state, or multi-turn history threading.
- `kanban.test.ts` covers `moveCard` but has zero direct tests for the `toCardId`/`fromCardId`/`toColumnId`/`fromColumnId` ID-prefixing round-trip helpers, despite these being load-bearing for every API call.

### Security

No XSS issues found — no `dangerouslySetInnerHTML` anywhere; all user content goes through JSX text interpolation (React auto-escapes). `.env` is correctly gitignored. The `X-User` header trust is a backend concern (see High finding #2 above), not a frontend defect.

---

## Docker / Scripts

- Multi-stage Dockerfile is correct: builds the frontend static export, copies only `out/` into the final Python image, uses `uv` per project convention, drops to a non-root user, and includes a `HEALTHCHECK`. No issues found.
- `scripts/start-*.sh` / `start-windows.ps1` don't pre-check that `.env` exists before passing `--env-file` to `docker run` — failure mode is a raw Docker error rather than a message pointing at `OPENROUTER_API_KEY`. Minor UX polish, not required for MVP.
- Stop scripts are minimal, correct, and appropriately simple for a single named container.

---

## Recommended next steps (in order)

1. Fix the static-file path traversal (`backend/app/routes/static.py`) — exploitable today, highest impact, smallest fix.
2. Decide the intended trust model for `X-User` and lock it down to `DEFAULT_USER` only, matching the stated "one board per user" MVP design.
3. Debounce column rename on the frontend.
4. Add backend tests for AI action dangling-reference handling and the uncaught `ValidationError` path; wrap `model_validate` in `chat.py`'s flow.
5. Add frontend tests for the board/chat error states and the card mutation rollback branches — currently the largest blind spot in frontend coverage.
6. Address the Medium/Low items opportunistically as related code is touched (duplicate `get_db`, position-clamp duplication, drag-and-drop targeting documentation).
