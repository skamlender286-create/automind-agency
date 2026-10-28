# AGENTS.md — AutoMind Agency (static site)

## What this is
Four **standalone HTML documents** with inline CSS/JS. No build step, no package
manager, no backend, no test runner, no framework. The pages are the app.

| Path | Tool |
| --- | --- |
| `index.html` | AI chat assistant + lead capture (the entry point on port 3000) |
| `research-agent-1.html` | E-commerce lead finder |
| `email-agent-1.html` | Cold-email generator |
| `auto-send-email-agent.html` | Bulk cold-email writer + send attempt |

## Running it
`docker compose -f docker-compose.base44.yml up -d` — a `python:3.12-slim` service
serves the checkout from disk on host port 3000. Nothing to install, nothing to
migrate, no volumes to preserve. The same files serve any edit on refresh.

## Non-obvious findings
- **The pages are separate documents, not routes.** There is no client-side router
  and no navigation between them. Reach each one by its full path
  (`/research-agent-1.html`, etc.); SPA-style pushState navigation will change the URL
  but leave the old page on screen. This is why the preview's `navigate` helper cannot
  open them.
- **No environment variables, no server-side secrets.** Nothing in the repo reads
  `process.env`/`getenv`; the app needs no credentials to boot or to serve. Do not
  declare secrets or wire `/run/base44/app.env` — no code would read them.
- **Every AI feature is keyed from the page itself.** The user pastes a Groq API key
  into that page's own setup panel, and it is kept in `localStorage`:
  `index.html` → `am_config` (a JSON blob with `groqKey`); `research-agent-1.html` →
  `groq_key`; `email-agent-1.html` → `groq_key_email`; `auto-send-email-agent.html` →
  `ams_gmailAddr`, `ams_gmailPass`, `ams_groqKey`.
- **All model calls go browser → Groq** at `https://api.groq.com/openai/v1/chat/completions`
  with model `llama-3.3-70b-versatile`. With no real key those paths return a Groq 401/error;
  the pages must be exercised with the user's own key, which is why end-to-end AI
  behaviour is not verifiable from a fresh checkout.
- **`auto-send-email-agent.html` cannot actually send mail.** Direct SMTP from a browser
  is blocked; the page generates the content and falls back to `mailto:`/copy. This is a
  design limit of a frontend-only app, not a setup problem.
- Page copy is a mix of English and Hinglish — that is intentional, not a bug.

## Verifying it works
```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:3000/   # and each page path
docker compose -f docker-compose.base44.yml ps                     # web should read (healthy)
```
Healthcheck in the compose file probes `/` over HTTP, which exercises the server itself.
In the preview, confirm `index.html` shows its "Agency Setup" panel, then save the form
(fields are local-only — no network call) and check the greeting bubble appears.
