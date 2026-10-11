# Review of PLAN.md and Working Tree

Date: 2026-10-11. Scope: `planning/PLAN.md` (including the uncommitted diff against `HEAD` 14550e1), plus the current git working tree.

Severity: **Critical** blocks work or loses data; **High** will cause bugs or test failures; **Medium** is an ambiguity agents will resolve differently; **Low** is polish.

---

## 1. Working tree: likely accidental deletions

`git status` shows 33 tracked files deleted (unstaged, about 5,400 lines) and the directories gone from disk: `backend/`, `planning/archive/`, `.github/workflows/`.

| # | Severity | Finding | Recommendation |
|---|---|---|---|
| 1.1 | **Critical** | The whole `backend/` tree is deleted: the finished market-data package (`app/market/*.py`), its tests (`tests/market/*`), `pyproject.toml`, `uv.lock`, `CLAUDE.md`, `README.md`, `market_data_demo.py`. `uv.lock` was added in the last commit ("Ready for Teams"), so deleting it right after committing looks unintended. `CLAUDE.md` still says "the market data component has been completed". | If this was not on purpose, run `git restore backend/`. If it was on purpose (a clean rebuild by agent teams), commit the deletion and update `CLAUDE.md` and PLAN.md to say the market data must be rebuilt. |
| 1.2 | **Critical** | `planning/MARKET_DATA_SUMMARY.md` and `planning/archive/*` are deleted, but the root `CLAUDE.md` (unchanged) sends agents to them. Agents will follow a broken reference. | Restore them with `git restore planning/MARKET_DATA_SUMMARY.md planning/archive/`, or remove the reference from `CLAUDE.md`. |
| 1.3 | High | `.github/workflows/claude.yml` and `claude-code-review.yml` are deleted, and `.github/` is now empty. This turns off the PR review automation. | Restore them unless removing CI is intended. |
| 1.4 | Medium | `.claude/settings.json` is deleted. It enabled the `frontend-design`, `context7` and `playwright` plugins. Playwright matters for the E2E work in section 12. | Restore it with `git restore .claude/settings.json`. |
| 1.5 | Low | New untracked files: `.claude/agents/reviewer.md`, `.claude/commands/doc-review.md`, `planning/STRETCH.md`. Typos: "feddback" in reviewer.md, "new new section" in doc-review.md. STRETCH.md is a sensible home for the cloud-deployment text that was removed from PLAN.md. | Fix the typos and commit. |

Suggested one-shot recovery, keeping the PLAN.md edits:
```
git restore backend/ planning/MARKET_DATA_SUMMARY.md planning/archive/ .github/ .claude/settings.json
```

---

## 2. PLAN.md diff conflicts with the completed market-data code (as of HEAD)

The diff makes the spec much tighter, which is good. Several new rules, though, contradict the code in `backend/app/market/` at HEAD. The plan does not mention that this code has to change. Since `CLAUDE.md` tells agents the market data is "completed", they will not change it.

| # | Severity | Plan says | Code at HEAD | Recommendation |
|---|---|---|---|---|
| 2.1 | High | `change_percent` is measured against `session_open`, and the payload includes `session_open`. | `PriceUpdate` has no `session_open`, and `change_percent` is measured against the previous tick (`models.py`). | Add a "Required market-data changes" subsection listing: store session open in `PriceCache`, add the field, change the `change_percent` formula, and update the tests. |
| 2.2 | High | `timestamp` is ISO 8601 (`"2026-10-11T14:30:00Z"`). | `to_dict()` sends Unix float seconds. | Pick one. ISO is fine, but the plan should say "change `to_dict()`". |
| 2.3 | Medium | Server sends a `: keepalive` comment every 15s. | `stream.py` sends nothing when the version is unchanged. | List this as a required change. |
| 2.4 | High | Tracked tickers = watchlist ∪ open positions. | `remove_ticker()` in both sources also calls `cache.remove()`. A watchlist DELETE handler that calls it naively drops the price of a held position, and then valuation and selling break. | State that the watchlist DELETE handler calls `source.remove_ticker()` only if no position exists. Likewise, when a sell closes a position, remove the ticker only if it is not on the watchlist. |
| 2.5 | Medium | (not covered) | `stream.py` defines `router` at module level, and `create_stream_router` adds routes to it, so calling it twice (for example, in tests) registers duplicates. | Create the `APIRouter` inside the factory. |
| 2.6 | Medium | Session open = first price seen. | `cache.remove()` on removal means a ticker that is removed and re-added gets a new session open, and in the simulator a new random seed for unknown tickers. | Accept this and document it, or keep a per-session open map that survives removal. |

---

## 3. Spec gaps and ambiguities

### Trading and data
| # | Severity | Finding | Recommendation |
|---|---|---|---|
| 3.1 | High | **LLM "add X and buy X" fails.** Watchlist changes run first, but a newly added ticker has no cached price until the next simulator tick (~500ms) or Massive poll (15s). The trade that follows returns "no price available". This is probably the most common demo request. | The simulator's `add_ticker` should write the seed price to the cache synchronously. Note in the plan that this case is expected to fail with Massive. |
| 3.2 | Medium | How to value a position with no cached price (Massive "pending") in `GET /api/portfolio`, in snapshots, and in the heatmap is not defined. | Suggest valuing at `avg_cost` (P&L 0) until a price arrives. |
| 3.3 | Medium | The order of rounding is unclear. Is cost rounded to cents before the `cash >= cost` check? Does `avg_cost` get rounded? | "cost = round(qty*price, 2); require cash >= cost; avg_cost stored unrounded." |
| 3.4 | Low | 500 snapshots at one every 30s covers about 4 hours. After a restart, the P&L chart shows only the last 4h. | Acceptable. Mention it so nobody "fixes" it. |

### API contracts
| # | Severity | Finding | Recommendation |
|---|---|---|---|
| 3.5 | High | "All errors return `{"detail": "human-readable reason"}`" conflicts with FastAPI defaults: request-validation failures (bad `side`, a string quantity) return **422** with `detail` as a **list**. The frontend will render `[object Object]`. | Either accept 422 list errors from validation, or add a `RequestValidationError` handler that flattens them to a string. State which. |
| 3.6 | Medium | Response bodies and status codes are missing for `POST /api/watchlist`, `DELETE /api/watchlist/{ticker}`, and `GET /api/health`. Duplicate add and remove-missing are also undefined (409? 404? idempotent 200?). | Suggest: both return `{"tickers": [...]}`; 400 for an invalid ticker, 409 for a duplicate, 404 for a missing ticker; health returns `{"status": "ok"}`. |
| 3.7 | Medium | Chat failure handling says "the broken response is not stored". It does not say whether the **user** message is stored. If it is, history ends up with an unanswered user turn. | "On failure, store neither message." |
| 3.8 | Low | It is not stated whether the `actions` JSON on past assistant messages goes into the LLM history. Including it helps the model avoid repeating trades. | Append a short action summary to assistant content in the history. |
| 3.9 | Low | The SSE example spreads one JSON object over two lines. In SSE a raw newline ends the `data:` field. | Add the note "(shown wrapped; sent as one line)". |

### Mock LLM
| # | Severity | Finding | Recommendation |
|---|---|---|---|
| 3.10 | Low | Substring matching with "first match wins" means "add BUYZ" or "remove SELL" trigger a trade. Also, "Buys 1 AAPL" assumes AAPL is priced, which holds only while AAPL is tracked. | Match whole words (`\bbuy\b`) and check the `add`/`remove` patterns before `buy`/`sell`. Fine for E2E as written. |

### Deployment and testing
| # | Severity | Finding | Recommendation |
|---|---|---|---|
| 3.11 | High | **E2E isolation.** The start script always mounts the named volume `finally-data`. On the second run, "Fresh start: $10k balance" fails because the old trades persist. | Let the start script take a volume name or a `--fresh` flag, or have E2E run `docker run` with a throwaway volume. |
| 3.12 | High | **How E2E enables `LLM_MOCK=true`** is undefined. The start script passes `--env-file .env`, which normally has `LLM_MOCK=false`. | Add `-e LLM_MOCK=true` passthrough to the start scripts (for example, a `--mock` flag). |
| 3.13 | Medium | `docker run --env-file .env` fails if `.env` is missing. `.env.example` does not exist in the repo yet. | Start scripts should copy `.env.example` to `.env` if it is missing. Make creating `.env.example` an explicit task. |
| 3.14 | Medium | The `DB_PATH` default "<repo>/db/finally.db" is ambiguous inside the container, where there is no repo root. `StaticFiles(directory="static")` depends on the working directory. | In the Dockerfile set `WORKDIR /app` and `ENV DB_PATH=/app/db/finally.db`. Locally, resolve the path relative to `backend/..`. |
| 3.15 | Medium | `.gitignore` has no entry for `db/finally.db` (only `db.sqlite3`), even though the plan says it is gitignored. `db/.gitkeep` does not exist either. | Add `db/*.db` and `db/*.db-*`, and create `db/.gitkeep`. |
| 3.16 | Low | "Reads `.env` from the project root": with `--env-file` the container needs no dotenv, but local `uv run` does. | Say: "local dev uses python-dotenv to load `../.env`; the container relies on `--env-file`." |

---

## 4. Consistency with other docs

- `README.md` still says "SQLite with lazy initialization" (the plan now says startup initialization), and its env table lacks `DB_PATH`. It also shows a raw `docker run` as Quick Start, while the plan says the scripts are "the only supported launch path". (Low)
- `CLAUDE.md` refers to deleted docs (see 1.2). (Critical)

---

## 5. Opportunities to simplify

- **Drop the 30s snapshot background task.** Record snapshots only after trades, plus one at startup. The P&L chart still works and one background task disappears. The cost: the line stays flat between trades, so it won't show market drift. That is a product call, but it is the biggest simplification on offer.
- **The `trades` table is never read by any endpoint.** Keep it as an audit log, but note that no API exposes it so nobody builds one.
- **Leave the `user_id` columns as they are.** They are cheap. But `users_profile.user_id` being the primary key while every other table uses a UUID `id` is fine and needs no change.
- **The diff already simplified a lot:** removing docker-compose, host-run Playwright, Vitest for pure logic only, explicit chart libraries, and moving the stretch goal out. These are good calls.

---

## 6. Summary of actions

1. Decide whether the `backend/`, `planning/archive/`, workflow and settings deletions were intended. If not, `git restore` them (Critical).
2. Add a "Required market-data changes" subsection covering session_open, ISO timestamp, keepalive, remove-vs-positions, and synchronous seeding on add (High).
3. Settle the 422 error shape, the missing watchlist and health contracts, and pending-price valuation (High/Medium).
4. Give E2E a mock flag and a fresh volume in the start scripts (High).
5. Fix `.gitignore` for `db/*.db`, add `db/.gitkeep` and `.env.example`, and set `DB_PATH` and `WORKDIR` in the Dockerfile (Medium).
