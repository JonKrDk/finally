# Review of uncommitted changes (2026-09-12)

Changes reviewed: `README.md` rewritten, `.claude/settings.json` gains a Stop hook, `.claude/agents/reviewer.md` added (untracked), previous `planning/REVIEW.md` deleted.

## README.md — accuracy against PLAN.md

Accurate on: all four env vars (§5), port 8000, `db/finally.db` / `/app/db`, `finally-data` volume, the `docker run` line (§11), local-dev commands (§10), model `openai/gpt-oss-120b` on Cerebras (§9), and the Status section scoping "complete" to `backend/app/market/`.

Findings (mostly the README promising things that do not exist yet):

1. **`README.md:30` — `cp .env.example .env` references a file not in the repo.** PLAN §5 says `.env.example` should be committed but it isn't. Add it, or reword to "create `.env` with `OPENROUTER_API_KEY=...`".
2. **`README.md:31-35`, `40-41` — Quick Start advertises `scripts/start_*`, `stop_*`, and a `Dockerfile` that do not exist.** No `scripts/`, `Dockerfile`, `frontend/`, `test/`, or `db/` yet; `docker build` fails immediately. Status (`README.md:82`) admits this, but readers hit Quick Start first. Add a "not yet runnable — see Status" caveat or move Status above Quick Start.
3. **`README.md:56-57` — Local Development references `app.main:app` and `frontend/`.** `backend/app/` contains only `market/`; no `main.py` yet.
4. **`README.md:63-65` — Tests.** Only `cd backend && uv run pytest` works today. `npm test` and `test/docker-compose.test.yml` don't exist, and PLAN §12 never specifies the E2E invocation — `docker compose … up --abort-on-container-exit` is an invention. Either move it into PLAN.md if authoritative, or omit until the file exists.
5. **`README.md:48` vs `:50` — `OPENROUTER_API_KEY` "Required: Yes" while `LLM_MOCK=true` needs "no API key".** Consider "Yes (unless `LLM_MOCK=true`)". Mirrors PLAN §5's own ambiguity.
6. **`README.md:51` — `FINALLY_DB_PATH` default shown as `db/finally.db`.** PLAN §5 states `<backend>/../db/finally.db`; correct relative to repo root only. Minor.
7. **`README.md:77` — tree omits `db/.gitkeep`, `Dockerfile`, `.env.example`.** Fine for conciseness.
8. **`README.md:82` — Status** is accurate; could mention the backend DB layer (PLAN §13 step 1). Nit.
9. **Line endings.** README.md written with LF on a CRLF-configured checkout; git warns it will normalize. Harmless.

## README.md — clarity

- Architecture table and single-line Quick Start are clear improvements over the previous version.
- `README.md:22` naming `/api/stream/prices` is the only endpoint mentioned; consider pointing to PLAN §8 instead. Nit.
- Linking `planning/PLAN.md` as the shared contract is a good addition.

## Other changes

10. **`.claude/settings.json:7-17` — Stop hook runs the reviewer subagent on every stop.** Runs a full review agent (rewriting `planning/REVIEW.md`) after every turn, including trivial ones: cost/latency concern, and verify the harness guards against `stop_hook_active` recursion since the hook itself edits files. If personal workflow, `settings.local.json` is a better home.
11. **`planning/REVIEW.md` deleted.** The previous file was a substantive review of PLAN.md with 11 issues, several still unresolved (curl in slim image healthcheck, missing `.env.example`, SSE atomicity, trade transaction/precision). Preserve it in `planning/archive/` if the hook is meant to regenerate this file.
12. **`.claude/agents/reviewer.md` is untracked.** If the hook in `settings.json` is committed, the agent definition must be committed too.

## Summary

The README is accurate on ports, paths, env vars, volume name, model, and component status. Main problem: Quick Start, Local Development, and Tests read as runnable but reference files that don't exist yet. Recommend adding `.env.example` now and a "not yet runnable" note near the top. The Stop-hook/reviewer-agent addition and the deletion of the prior REVIEW.md are unrelated to the README rewrite and would be cleaner as a separate commit.
