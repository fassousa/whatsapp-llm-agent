# Architecture

## Components

| Component | Responsibility | Failure if missing |
|---|---|---|
| `bot.js` | Socket lifecycle, router, handlers, scheduler | nothing works |
| `lib/text.js` | Command parsing, normalization, URL extraction | wrong command routing |
| `lib/dedupe.js` | Prevents processing the same message twice after reconnect | duplicate replies and duplicate alerts |
| `lib/resets.js` | Domain date math (weekly/fortnightly resets) | wrong countdowns |
| `lib/authCleanup.js` | Purges stale session material | session bloat, slower pairing |
| `watchdog.js` | External supervision with heartbeat | silent death goes unnoticed |
| `game_data.json` | Versioned domain data (patch, banners, events) | stale answers |

## Message flow

```
1. Baileys emits messages.upsert
2. Filter: is it a group? is the group allowed? (ID or name)
3. Extract: text, media type, quoted message, sender candidates (LID/JID/9th digit)
4. Parse: prefix "!" required, normalize alias → canonical command
5. Authorize: public | admin | owner
6. Handle:
   a. media → download → convert (sticker/GIF) or send to model (audio/image)
   b. text  → static answer, cached domain data, or formatted report
7. Reply (quoted when applicable)
8. Update counters and caches (atomic write)
```

Every step that can fail is wrapped: a handler error logs and returns, it never takes the socket down.

## Reconnect state machine

```
connected ──disconnect(code)──► classify(code)
                                  │
        ┌─────────────────────────┼──────────────────────────┐
        ▼                         ▼                          ▼
   401 loggedOut            515 restartRequired         428 / 500 / other
   stop, require pairing     reconnect ~1s              backoff (500: ≥60s)
        │                         │                          │
        └─────────────────────────┴──────────────► reconnecting ──► connected
```

State (`attempts`, last status) is persisted in `.reconnect_state.json` so a process restart does not reset the backoff into a hot loop.

## Liveness probe

```
every PROBE_INTERVAL_MS (default 120s):
  fetchStatus(target) with PROBE_TIMEOUT_MS (default 20s)
    ok        → failures = 0
    timeout   → failures++
    failures ≥ PROBE_MAX_FAILURES (3) → controlled reconnect, failures = 0
```

The probe never restarts the process — that is the watchdog's job. Separation keeps responsibilities testable.

## Watchdog

```
every 60s:
  pm2 describe bot → is it online?
  .heartbeat mtime  → is it fresh (< WATCHDOG_STALE_MS, default 5 min)?
  if not, and last restart > 5 min ago, and not intentionally stopped:
     pm2 restart bot  (+ optional Telegram alert)
```

`WATCHDOG_DRY_RUN=1` and `node watchdog.js --check` exist so the logic can be exercised without touching production.

## Persistence

| File | Content | Why a file |
|---|---|---|
| `auth_info*/` | Pairing/session material | required by Baileys, must survive restarts |
| `.reconnect_state.json` | attempts, last status | avoid hot-loop after crash |
| `.heartbeat` | last successful loop timestamp | cheap liveness signal for the watchdog |
| `codes_cache.json` | seen redeem codes | avoid duplicate alerts |
| counters | usage stats | survive restarts without a DB |

All writes are atomic (write temp + rename) so a kill -9 cannot leave a half-written JSON that breaks startup.

## Failure modes and mitigations

| Failure | Symptom | Mitigation |
|---|---|---|
| Session invalidated | `401` on connect | stop and require re-pairing; never loop |
| Socket half-open | no messages, process alive | probe → controlled reconnect |
| Process wedged | PM2 says online, no heartbeat | watchdog restart |
| Duplicate delivery after reconnect | same command answered twice | dedupe by message ID |
| Model quota exhausted | handler throws | fallback chain + user-facing message |
| Bad deploy | syntax error in prod | `node --check` + JSON validation before `scp` |
| Log leak | session keys in logs | redaction before write |

## Observability

- Every log line has an ISO timestamp and a subsystem tag (`[PROBE]`, `[AGENDADOR]`, `[GEMINI]`).
- `!status` exposes RAM, CPU, network, uptime and usage counters from inside the chat — the cheapest possible dashboard for a bot whose operator is also a user.
- Health is observable from three places: PM2, the heartbeat file, and the chat itself. If all three are silent, the watchdog alerts.
