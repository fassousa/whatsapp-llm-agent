# Decisions

Short ADRs for the choices that shaped the project. Each one lists the alternative I rejected and what it cost me.

---

## ADR-001 — Baileys over the official Cloud API

**Context.** The bot serves a community on a personal number, answering free-form commands in groups.

**Decision.** Use Baileys (unofficial, multi-device).

**Why.** The Cloud API requires a business number, template messages and per-conversation pricing — it cannot answer arbitrary commands in an existing community group. Baileys gives full protocol access at zero marginal cost.

**Consequences.** No delivery guarantees, no SLA, session can be invalidated at any time, and the account is exposed to platform policy. This is the root cause of most of the reliability work (probe, watchdog, reconnect state machine) — the architecture is, in large part, a compensation for this decision.

---

## ADR-002 — JSON state instead of a database

**Context.** Counters, caches and quotas need to survive restarts. The workload fits comfortably in memory.

**Decision.** JSON files with atomic writes; no database.

**Why.** One process, one VM, no operational budget. Adding Postgres would have doubled the infrastructure for state that is measured in kilobytes.

**Consequences.** No concurrent writers, no transactions, no queries. Moving to multi-process or multi-tenant means migrating state first. I would start with SQLite next time — same operational simplicity, real transactions.

---

## ADR-003 — Gemini Flash with a model fallback chain

**Context.** Audio transcription, image description and summarization, on a free-tier key.

**Decision.** Call Gemini with an ordered list of candidate models, falling through on quota/availability errors, and surface the error in plain Portuguese when all fail.

**Why.** Free-tier quota is exhausted unpredictably; a hardcoded model turns a transient quota error into a broken feature. The fallback chain is ~20 lines and removed a whole class of failures.

**Consequences.** Slight behavioral variance between models; latency depends on which model answers. Acceptable for transcription and summaries, not acceptable for anything that must be deterministic.

---

## ADR-004 — PM2 + custom watchdog instead of an orchestrator

**Context.** The process must stay alive without human attention.

**Decision.** PM2 for process supervision plus a separate watchdog that checks an application-level heartbeat.

**Why.** PM2 restarts a *dead process*; it does not notice a *live process with a dead socket*. The heartbeat adds the only check that actually matters. Kubernetes would solve it, at a cost of infrastructure the project cannot justify.

**Consequences.** Two supervision mechanisms to reason about. The watchdog must be careful to distinguish "crashed" from "intentionally stopped", or it fights the operator.

---

## ADR-005 — Configuration as environment, not code

**Context.** The first version had owner numbers, guest IDs and group IDs hardcoded.

**Decision.** Move every deployment-specific value to `.env`, validated at startup (fail fast when `BOT_OWNERS` is empty).

**Why.** Three reasons, in order of importance: (1) hardcoded personal data makes the repository unpublishable; (2) config changes required a code edit and a full deploy; (3) nobody could audit who had access without reading the source.

**Consequences.** A bad `.env` is now a startup failure instead of a silent misconfiguration — which is the desired behavior for an authorization-critical setting. It also made this case study possible.

---

## ADR-006 — Prefix commands instead of natural language

**Context.** The bot lives in busy group chats where people talk constantly.

**Decision.** Every command requires the `!` prefix; natural-language triggers are not accepted for anything that writes or spends money.

**Why.** A bot that reacts to any message becomes noise, and an LLM-routed bot in a group chat is an unbounded cost and prompt-injection surface.

**Consequences.** Slightly worse UX for new users (mitigated by `!ajuda`). In exchange, the bot never speaks unless spoken to — the single most important property for staying welcome in a community.

---

## What I would change if I rebuilt it

1. **SQLite instead of JSON** — same footprint, real consistency.
2. **A queue for model calls** — decouples latency and cost from user behavior.
3. **Tests on the authorization matrix** — the highest-risk code currently has the least coverage.
4. **A plugin boundary for domain commands** — the game layer should be a module, not a section of `bot.js`.
