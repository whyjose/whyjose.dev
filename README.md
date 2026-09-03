# whyjose.dev

Source for [whyjose.dev](https://whyjose.dev). The front page is a chatbot (**JOSÉ.EXE**) that answers questions about me, backed by a Cloudflare Worker sitting in front of an LLM.

`worker.js` isn't in this repo. It's got the actual system prompt in it, and that prompt tells the bot to never reveal itself — kind of ruins the bit if I also put the file on GitHub. It lives in a private repo instead. Everything about how it works is documented below anyway. Want to actually see the code? Ask JOSÉ.EXE how to contact me: [whyjose.dev](https://whyjose.dev)

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
| Frontend | Vanilla HTML/CSS/JS | no build step, nothing to keep patched |
| Hosting | GitHub Pages | free, custom domain, done |
| Edge proxy | Cloudflare Worker | keeps the API key off the client, runs close to the user, free tier |
| Model routing | OpenRouter | one API, so one model having a bad day isn't an outage |

### Files in this repo

```
index.html      chat UI + client logic, theme toggle
about.html      the human-written about page
404.html        branded not-found page
CNAME
```

`worker.js` and `wrangler.toml` live in the private companion repo. Deploy is manual (`wrangler deploy`) — no CI, on purpose, see below.

---

## Endpoints

| Method | Path | Behaviour |
|---|---|---|
| `POST` | `/` | chat. origin-checked, rate limited, proxied to OpenRouter |
| `OPTIONS` | `/` | CORS preflight, origin-checked |
| `GET` | `/health` | `200 {"status":"ok","ts":…}` — answered before the origin check and rate limiter, never calls OpenRouter, so checking it costs nothing |

---

## Security stuff, and what it actually buys

| Decision | What it actually protects against |
|---|---|
| API key as a Worker secret | never in client JS, never in git. the one that actually matters |
| System prompt server-side | can't read or edit the persona from view-source |
| Origin allowlist | stops browser-based cross-origin calls. not authentication — see below |
| Per-IP rate limit (15/hour) | the real backstop against someone scripting my OpenRouter bill up |
| Input caps | message capped at 500 chars, history trimmed to 10 turns, unknown fields dropped before anything hits the model |
| Replies as `textContent` | no `innerHTML`, no `eval` — a model can't inject markup into its own reply |
| `X-Content-Type-Options: nosniff` | no MIME sniffing on Worker responses |
| Client-side `AbortController` (28s) | `fetch` has no default timeout, so without this a stalled connection just hangs forever |

**On the origin check specifically:** it only works because a browser is honest about the `Origin` header. Something that isn't a browser is not. `curl -H "Origin: https://whyjose.dev" ...` gets through fine — checked that against the live endpoint myself, didn't just assume. So it stops another site embedding a script that spends my OpenRouter credits from someone's browser. It does not stop anyone hitting the endpoint directly. The rate limit is what actually caps that.

---

## The model fallback chain

18 models, paid ones first, free ones as the safety net. If one 429s, 5xxs, or times out, it moves to the next. User never sees a model-level error.

Two things I had to fix so this didn't become its own problem:

- **Auth errors stop the loop instead of grinding through it.** Every model uses the same key, so if OpenRouter returns a `401`, all 18 will. Used to just try the next one anyway — pure wasted time, and it turns "my key expired" into a misleading "all models unavailable." Now it breaks on the first `401`/`403` and logs it as what it actually is.
- **There's a time budget.** 18 models × 15s each is a ~4.5 minute worst case if things go badly. The loop now tracks how long it's been running and stops trying new models past 20s. Client gives up at 28s.

---

## Model quirks

Some models leak their own thinking into the reply — `<think>` tags, "okay so the user is asking" type stuff. That gets stripped.

Used to also strip anything starting with common words like "First" or "Note:", which occasionally ate a real reply that just happened to start that way, sometimes leaving nothing at all, which then got mistaken for a content-filter block downstream. Fixed to only match actual reasoning phrasing, and if stripping would empty the reply, it just keeps the original instead.

---

## What's not great yet

Roughly in order of how much it'd bother me if this were a real product and not a portfolio piece:

- **Rate limiter isn't actually global.** It's a plain `Map` living inside a Worker. Cloudflare runs a bunch of isolates across a bunch of locations and recycles them whenever, so "15 per IP per hour" is really "15 per IP per hour per isolate that happens to catch you," and it resets when that isolate gets evicted. Fine for a personal project with a capped budget. Wouldn't ship it like this for anything that actually spends real money.
- **Origin check doesn't stop a script, only a browser.** Covered above. A signed token per page load or a Turnstile challenge would close it — more moving parts than this needs right now.
- **Deploy used to be copy-pasting into the Cloudflare dashboard.** No diff, no rollback, no way to tell if the deployed code matched the repo. `wrangler.toml` fixed that part.
- **No CI.** Deploys are manual, nothing polls `/health` on its own. Didn't feel worth wiring up automation for one endpoint, especially since I've already got that pattern shown elsewhere.
- **No real logging.** Just `console.log` and Worker tail. Can debug live, can't tell you how often the chain actually falls past model 5.
- **No tests.** The reasoning-strip bug above is a decent argument I should add some.
- **`resetAt` on the 429 is a best guess, not a promise** — since the counter itself isn't exact either.

---

*Built by José. Maintained against its will by JOSÉ.EXE.*
