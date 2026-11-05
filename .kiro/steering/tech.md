---
inclusion: always
---
# Tech stack

- **Language:** TypeScript (strict mode) everywhere.
- **Frontend:** Vite + React, built as a static site.
- **Fetch proxy:** one small Cloudflare Worker that fetches public URLs for Site Séance, so the browser avoids CORS limits. It does nothing else.
- **Core checks:** a pure TypeScript package with no network or DOM access. It takes fetched text in and returns typed results out.
- **Tests:** Vitest for unit tests, fast-check for property-based tests.
- **Hosting:** Cloudflare free tier only (`*.pages.dev` / `*.workers.dev`). No paid plans, no custom domain.
- **Package manager:** npm workspaces.

## Rules
- Prefer small pure functions with explicit input and output types.
- Parsers (robots.txt, JSON-LD, HTML text extraction) get property-based tests, not only examples.
- robots.txt matching follows RFC 9309: longest matching path wins; on a tie between Allow and Disallow, Allow wins; user-agent groups match case-insensitively.
- The Haunting Score must be bounded (0–100) and monotonic: fixing a failing check can never lower the score.
- No dependency that needs a paid licence or an account.
