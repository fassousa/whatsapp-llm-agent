# WhatsApp agent with an LLM in the loop — an engineering case study

A production WhatsApp assistant that has been running unattended for months on a single GCP VM, serving a real community: stickers and GIFs, audio transcription and summarization, image description, group administration, and a daily job that checks for new redeem codes and posts them at 09:00 BRT.

The code is private because it serves real users (phone numbers, group IDs and session material). This repository documents the engineering: the constraints, the architecture, the failure modes and the trade-offs. Excerpts available on request.

---

## 1. The constraint that shapes everything

WhatsApp has no official bot API for personal numbers. The practical options are:

| Option | Reality |
|---|---|
| WhatsApp Cloud API | Official and stable, but tied to a business number, template messages and per-conversation pricing — wrong fit for a community bot answering free-form commands |
| Unofficial multi-device libraries (Baileys) | Full protocol access, no per-message cost, but **no delivery guarantees, no SLA, session can be invalidated by the server** |

I chose Baileys. That decision moves the entire engineering problem from "build features" to **"keep a fragile socket alive and never lose the session"** — which is where most of the work actually went.

## 2. Architecture

```
WhatsApp ──► Baileys socket ──► router (prefix + allowlist + owner check)
                                   │
                 ┌─────────────────┼──────────────────┐
                 ▼                 ▼                  ▼
           media pipeline     Gemini Flash       state layer
        (WebP/EXIF, GIF)   (transcribe/describe/  (JSON caches,
                            summarize)             counters, quotas)
                 │                 │                  │
                 └─────────────────┴──────────────────┘
                                   ▼
                        liveness probe (2 min)
                                   ▼
                   reconnect / watchdog restart (PM2)
```

Five components, one process, plus a second process for the watchdog. No queue, no database, no orchestrator — deliberate, because the operating budget is a single small VM and the workload is bursty but tiny.

## 3. Reliability is the product

A bot that dies silently is worse than no bot: users assume it works and get no answer. The reliability stack has three independent layers, because each one catches a failure the others miss.

**Layer 1 — reconnect logic inside the process.** Baileys reports disconnect reasons as status codes, and the correct response differs per code:

| Code | Meaning | Response |
|---|---|---|
| `515` | restart required to finish pairing | reconnect in ~1 s — waiting 30–60 s here lets a freshly paired session expire and turn into `401` |
| `500` | bad session, server still cleaning the stream | wait at least 60 s |
| `428` | connection closed, often transient | normal backoff |
| `401` | logged out | do **not** loop; require a new pairing |

Getting `515` wrong is the classic way to lose a session right after pairing. The delay logic is a small pure function with its own test.

**Layer 2 — liveness probe.** The socket can stay "open" while the account is unreachable. Every 2 minutes the process calls `fetchStatus` with a timeout; after 3 consecutive failures it forces a controlled reconnect. This converts a silent hang into a recoverable event.

**Layer 3 — external watchdog.** A separate PM2 process checks every 60 seconds that (a) the bot process exists, (b) it is online according to PM2, and (c) a `.heartbeat` file is fresh (under 5 minutes). It restarts at most once per 5 minutes and respects an intentional `pm2 stop`. It also sends an optional Telegram alert.

The three layers are independent on purpose: a bug in the reconnect path cannot disable the probe, and a bug in the probe cannot disable the watchdog.

## 4. The media pipeline

WhatsApp users send whatever they have. The bot must normalize before it can do anything useful:

- **Stickers:** validate the WebP container (RIFF header check), inject EXIF metadata for pack name and author, convert images and short videos (≤ 6 s), and convert back to GIF on request.
- **Audio:** download the voice note, pass it to a multimodal model with an explicit mode (`transcribe` vs `summarize`), and return text. The two modes matter: a 4-minute audio summarized is cheaper and more useful than a wall of transcription.
- **Images:** describe, not classify — the prompt is tuned for "what is in this image, in Portuguese, for a chat context".
- **Quoted messages:** commands often arrive as a reply to the target media, so sender and media resolution must walk the quoted-message tree.

## 5. Cost control

LLM calls are the only marginal cost. Two rules keep it bounded:

1. **Expensive paths are owner-only.** Transcription and summarization are gated to the bot owner and a small allowlist of private-chat guests; groups never trigger them.
2. **Model fallback instead of failure.** The call tries a list of models in order and surfaces quota errors to the user in plain language instead of crashing the handler.

A future improvement is a queue with per-user budget, so cost is predictable rather than reactive.

## 6. Security model

A WhatsApp bot is an account with a phone number; the threat model is unauthorized use of that account and its data.

- **Two authorization primitives only:** owner list and group allowlist. Anywhere else, the bot is silent — it does not answer, react or log message content.
- **Sender resolution across ID variants.** WhatsApp now exposes LID and JID identifiers, and Brazilian numbers have an optional ninth digit. Authorization resolves the sender across all candidate IDs *before* deciding — the usual hole in WhatsApp bots is comparing a single ID.
- **Secrets and session material never leave the machine.** `.env` and `auth_info*` are gitignored; session keys are redacted from logs before being written.
- **Configuration is data, not code.** Phone numbers, group IDs and guest lists are environment variables; the repository contains no personal data. This was a deliberate refactor — the first version had them hardcoded, which made the code unpublishable and the config un-auditable.

## 7. Deployment

```
deploy.sh ──► node --check bot.js && JSON parse game_data.json
          ──► gcloud compute scp bot.js + game_data.json → VM
          ──► pm2 restart
```

The validation step exists because a syntax error in a 2,000-line file is easy to introduce and expensive to discover in production: the script refuses to deploy a broken file. Logs are rotated by `pm2-logrotate` (20 MB, 7 files, daily) and every line carries an ISO timestamp.

## 8. What I would do differently

- **State in a database from day one.** JSON-on-disk was right for a single process; it makes multi-tenant or multi-process deployments impossible.
- **Test the router, not just the pure helpers.** The pure modules have tests; the authorization matrix — the part where a bug is a security bug — does not.
- **Queue the expensive calls.** Synchronous LLM calls inside the message handler make latency and cost coupled to user behavior.
- **Keep the domain layer out of the engine.** Game-specific commands and data are entangled with generic features; a plugin boundary would have made both cleaner.

## 9. Stack

`Node.js (ESM)` · `Baileys` · `Gemini Flash` · `axios` · `pino` · `PM2` · `GCP Compute Engine` · `bash` tooling · `node --test` + GitHub Actions CI

---

*Related: [butler-bot](https://github.com/fassousa/butler-bot) (private) — the implementation. Architecture details in [docs/architecture.md](docs/architecture.md); decisions in [docs/decisions.md](docs/decisions.md).*
