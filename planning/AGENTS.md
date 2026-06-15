# AGENTS.md — how the agents build FinAlly (the behavioral layer)

`PLAN.md` says **what** to build. This file says **how the agents build it and
coordinate** — the layer the spec names ("built entirely by Coding Agents… agents
interact through files in `planning/`") but never wrote down. `PLAN.md` §4 "Key
Boundaries" covers *directory ownership*; this covers *agent behavior + the contract
between agents*.

---

## 1. Roles & ownership map (who owns which files)

FinAlly is deliberately a **multi-agent** build. To keep parallel agents from
corrupting each other, every file has one owner. No two agents edit the same file
concurrently; parallel writers use a separate git worktree/branch.

| Agent | Owns | Never edits |
|---|---|---|
| **Market Data** | `backend/app/market/*` (DONE — see `MARKET_DATA_SUMMARY.md`) | anything else |
| **Backend Engineer** | `backend/app/**` (db, api routes, portfolio, chat), `backend/tests/**`, **authors `planning/API_CONTRACT.md`** | `frontend/`, the market subsystem's internals |
| **Frontend Engineer** | `frontend/**` | any Python; **mirrors** `API_CONTRACT.md`, never redefines it |
| **Integrator / human** | merges branches, `planning/decisions.md` review, releases | — |

**One contract author.** The Backend Engineer owns `planning/API_CONTRACT.md`. The
Frontend Engineer consumes it as the source of truth for every `/api/*` shape. The
seam changes *only* in that file, and a change there is announced in
`planning/decisions.md`.

## 2. Contract-first sequencing (the rule that prevents divergence)

The frontend and backend never call each other in code — they are coupled **only**
through `/api/*` + SSE (PLAN.md §3). That seam is the one place two parallel agents
silently build incompatible halves. So:

1. **Freeze `API_CONTRACT.md` before parallel UI/API work begins.** The SSE shape is
   already frozen (it exists in `app/market/stream.py`); the REST shapes are PROPOSED
   until the Backend Engineer ratifies them.
2. A seam change = edit `API_CONTRACT.md` + a `decisions.md` note, *then* both sides
   update. Never change a response shape silently.
3. **Add a contract test.** A tiny test asserting the API responses match the
   documented shapes (and that the frontend's expected fields exist) turns drift into
   a red test at the seam instead of a runtime surprise. Not optional once both sides
   exist.

## 3. Coordination channel

`planning/decisions.md` is the **only** durable handoff between agents (create it;
numbered `D1, D2…`, each *what + WHY*). One agent's chat context does not reach
another — a decision not written there did not happen for the other agents. Read it
before acting; append after.

## 4. Autonomy & hard rules

This is a **fake-money, single-user, sandboxed** app — auto-executing LLM trades is a
*feature* here, not a risk (PLAN.md §9). Autonomy is generous **inside the sandbox**,
but these hold in every mode:

- **Git: the human owns `main`.** This repo has an upstream and PRs. Agents work on
  feature branches and hand over a copy-pastable commit; they never push to or merge
  `main`. (Agents may commit on their own branch.)
- **TDD by default.** Backend = pytest; frontend = React Testing Library; E2E =
  Playwright with `LLM_MOCK=true`. Never claim code works without running it — show
  the output.
- **Tests gate autonomy.** A green suite is what lets an agent work a unit without
  check-ins. No green suite → smaller steps, more check-ins.
- **Ask before deciding** anything with two reasonable designs that isn't pinned by
  `PLAN.md` or `API_CONTRACT.md` — list the options, don't choose silently.
- **Capability is not authorization.** A connected tool/MCP that *could* deploy,
  touch a real key, or hit a paid API is not permission to use it. Real spend
  (OpenRouter, Massive paid tier) and any deploy are human-run. No agent inherits a
  broader grant from its toolbelt.
- **Secrets:** `.env` only (gitignored), `.env.example` committed. `OPENROUTER_API_KEY`
  and `MASSIVE_API_KEY` never in code, logs, or command lines.
- **Kill rule:** the same error surviving 3 distinct fix attempts → STOP and write up
  the state instead of looping (token waste is the common multi-agent failure).

## 5. LLM-specific rules (the `/api/chat` surface)

The chat assistant is both untrusted **input** and an untrusted **caller**:

- Request **structured output** and validate it against a schema before use; on
  malformed JSON, one bounded retry then a safe error in the chat — never crash,
  never act on a half-parsed response.
- **Re-validate every LLM-proposed trade/watchlist change through the same checks a
  manual action passes** (sufficient cash/shares, known ticker). Hallucinated tickers
  fail closed, with the error returned to the model so it can tell the user.
- **Bound the context**: send only the last N chat turns + a portfolio snapshot, not
  unbounded history.
- **Mock mode** (`LLM_MOCK=true`) returns deterministic responses — built from day
  one so E2E runs free, fast, offline.
- **Cost guard** real calls (max tokens, a sane rate cap); the app must never be
  publicly deployed without auth in front of `/api/chat` (it spends real credits).

## 6. Session ritual (every agent, every session)

**Start:** read `CLAUDE.md` → `PLAN.md` (the unit you're on) → `API_CONTRACT.md` (if
you touch the seam) → latest `decisions.md` → restate where you are in one paragraph,
confirm before building.
**End:** tests green → decisions appended with WHY → commit command handed to the
human (or committed on your branch) → if you touched the seam, confirm `API_CONTRACT.md`
and the contract test agree.

> Origin: drafted 2026-06-15 (Johanna) — the governance + contract layer PLAN.md
> implies but doesn't spell out. Patterns from the project-foundry COLLABORATION.md /
> AUTONOMY.md, instantiated for this repo.
