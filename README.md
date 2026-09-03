# whyjose.dev

Source for [whyjose.dev](https://whyjose.dev) — a static personal site whose front page is an LLM-backed chat persona (**JOSÉ.EXE**) that answers questions about me.

This repo holds the frontend that's actually deployed. The Cloudflare Worker that proxies chat requests to an LLM — the part with the interesting engineering — lives in a private repo. That's on purpose, not an oversight: the Worker's source includes the literal system prompt driving the persona, and publishing that verbatim would also publish the line in it telling the bot the prompt is confidential, which defeats itself the moment the file is public. The architecture and the reasoning behind every decision in it are documented in full below regardless. If you want to see the actual code, ask — [hey@whyjose.dev](mailto:hey@whyjose.dev) — and I'll walk you through it directly.

---

## Architecture

```
Browser  ──POST─▶  api.whyjose.dev            ──▶  openrouter.ai
(GitHub Pages)     (Cloudflare Worker)              /api/v1/chat/completions
                     │                                    │
                     │ • origin check                     │ routes to whichever
                     │ • per-IP rate limit                │ model is available
                     │ • input caps + history trim        │
                     │ • system prompt injection          ▼
                     │ • reasoning-leak stripping   ◀── reply
                     ▼
                   JSON { reply }
```

| Layer | Choice | Why |
|---|---|---|
| Frontend | Vanilla HTML/CSS/JS | No build step, no dependencies, nothing to keep patched |
| Hosting | GitHub Pages | Free static hosting on a custom domain |
| Edge proxy | Cloudflare Worker | Keeps the API key server-side; runs close to the user; free tier |
| Model routing | OpenRouter | One API across many providers, so a single model outage isn't an outage |

### Files in this repo

```
index.html      chat UI + client logic, theme toggle
about.html      the human-written about page
404.html        branded not-found page
CNAME
```

`worker.js` and its `wrangler.toml` deploy config live in the private companion repo, kept in sync with what's actually deployed. Deploys are manual (`wrangler deploy`) — no CI here, deliberately, see below.

---

## Endpoints (served by the Worker, documented here even though the code isn't)

| Method | Path | Behaviour |
|---|---|---|
| `POST` | `/` | Chat. Origin-checked, rate limited, proxied to OpenRouter. |
| `OPTIONS` | `/` | CORS preflight, origin-checked. |
| `GET` | `/health` | `200 {"status":"ok","ts":…}`. Answered before the origin check and rate limiter, and never calls OpenRouter — so monitoring is free and can't consume anyone's quota. |

---

## Security decisions, and what each one actually buys

| Decision | What it actually protects against |
|---|---|
| API key as a Worker secret | The key is never in client JS, never in git. This is the one that matters. |
| System prompt server-side | Visitors can't read or edit the persona by viewing source. |
| Origin allowlist | Restricts direct **browser-based** cross-origin use. It is not authentication — see below. |
| Per-IP rate limit (15/hour) | The real backstop against scripted abuse running up an OpenRouter bill. |
| Input caps | Message capped at 500 chars, history trimmed to 10 turns and 1000 chars per part, unknown fields dropped before anything reaches the model. |
| Replies rendered as `textContent` | No `innerHTML`, no `eval` — model output can't inject markup. |
| `X-Content-Type-Options: nosniff` | No MIME sniffing on Worker responses. |
| Client-side `AbortController` (28s) | `fetch` has no default timeout; without this the UI can hang forever on a stalled connection. |

### On the origin allowlist specifically

The `Origin` header is set by browsers, not by the server. A browser will honestly report where a page was loaded from, which is what makes the check useful: it stops another website from embedding a script that spends my OpenRouter credits.

It does **not** authenticate the caller. `curl -H "Origin: https://whyjose.dev"` sails straight through, and I verified that against the live endpoint rather than assuming. Anything describing this as "prevents the endpoint being called by third parties" is overselling it. The per-IP rate limit is what actually bounds abuse; the origin check is a cheap filter in front of it.

---

## Resilience: the model fallback chain

Requests are tried against an ordered list of 18 models — paid first (better persona consistency), free models as the safety net. If a model 429s or 5xxs or times out, the next one is tried. The user never sees a model-level error.

Two things keep that from becoming its own failure mode:

**Auth errors short-circuit.** A `401`/`403` from OpenRouter is an account-level problem — every model in the chain uses the same key, so retrying the other 17 is pure latency and it misreports "expired key" as "all models unavailable." Those statuses break the loop immediately and log distinctly rather than blending into the generic fallback noise.

**There's a total time budget.** 18 models × a 15s per-request timeout is a ~4.5 minute worst case. The loop tracks elapsed wall-clock time, stops starting new attempts past a 20s budget, and clamps each remaining request's timeout to whatever is left. The client aborts at 28s, leaving the Worker margin to answer first.

---

## Handling model quirks

Some models leak chain-of-thought into the response. `<think>…</think>` blocks are stripped, and so are lines that read as reasoning preamble.

That second filter used to match bare sentence openers — "First", "Note:", "Let me" — which silently deleted legitimate replies that happened to start "First of all, José…", sometimes emptying the reply entirely, which then got misclassified downstream as a content-filter block. It now matches chain-of-thought *phrasing* specifically, and if stripping would leave nothing behind, the original reply is kept instead. One leaked reasoning line is a far smaller failure than a correct answer replaced with a false moderation message.

---

## Known limitations / what I'd do next

The honest list, roughly in order of how much they'd bother me in production:

- **The rate limiter is per-isolate, not global.** The request counter is an in-memory map inside the Worker. Cloudflare runs many isolates across many colos and recycles them freely, so "15 requests per IP per hour" is really "15 per IP per hour *per isolate that happens to serve you*," and the counter resets whenever an isolate is evicted. Effective ceiling is higher than the number suggests, and it's not durable. The correct fixes are Cloudflare's rate-limiting binding or a Durable Object keyed by IP — the DO gives a single authoritative counter at the cost of a round trip per request. For a personal site with a capped OpenRouter budget the current version is good enough; I'd not ship it as-is anywhere with a real spend limit.
- **The origin check doesn't authenticate non-browser clients.** Covered above. A signed short-lived token minted per page load, or a proof-of-work / Turnstile challenge, would raise the floor. All three are more moving parts than this project currently justifies.
- **Deployment was a dashboard paste until recently.** The Worker lived only in the Cloudflare editor, which meant no review, no diff, no rollback, and a real chance the deployed code and the repo drifted. `wrangler.toml` (in the private repo, alongside the source) fixed that — deploys are `wrangler deploy` now, not a paste.
- **No CI/CD, on purpose.** There's a `/health` endpoint but nothing polls it automatically, and deploys are manual. Automating build/deploy/monitoring for a single personal endpoint isn't worth the upkeep here, and it's a pattern I've demonstrated elsewhere rather than duplicating on this project too.
- **No structured logging or tracing.** Failures go to `console.log` and Worker tail. Enough to debug interactively, not enough to answer "how often does the chain fall past model 5?" A counter per model outcome pushed to Analytics Engine would be the cheap upgrade.
- **No test suite.** The Worker's pure-ish pieces (history sanitisation, reasoning stripping, rate-limit window maths) are exactly the kind of logic that deserves unit tests. The reasoning-strip bug above is the direct argument for it.
- **`resetAt` is best-effort.** The 429 response returns the window's reset timestamp so the UI can say "come back in N minutes" — but since the counter is per-isolate, that timestamp is a hint, not a guarantee.

---

*Built by José. Maintained against its will by JOSÉ.EXE.*
