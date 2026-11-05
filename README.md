# GhostCheck 👻

**Is your brand a ghost to AI assistants?** Paste a URL and GhostCheck shows how readable your site is to AI crawlers and AI answer engines, with a 0–100 Haunting Score where every point links to its evidence.

Built for [Kiroween 2026](https://kiroween.devpost.com/) — Costume Contest category — with [Kiro](https://kiro.dev/).

## Status
🚧 **Spec phase.** Steering and requirements are in [`/.kiro`](./.kiro). Design, tasks and code are generated in Kiro from these specs. See [docs/KIRO_USAGE.md](./docs/KIRO_USAGE.md) for how Kiro is used and [KIRO_LOG.md](./KIRO_LOG.md) for the day-by-day log.

## Features (planned V1)
- **Site Séance** — no key, no sign-in: AI crawler rules in robots.txt (RFC 9309), `/llms.txt`, schema.org JSON-LD, text visible without JavaScript, basic meta tags.
- **AI Séance** — bring your own API key, or replay the committed demo sample: ask buying-intent questions and see whether your brand is mentioned.
- **Haunted UI** — the ghost becomes visible as your site becomes readable to AI.

## Principles
- No secrets in this repo or the deployed app. API keys stay in your browser tab.
- No customer data. Demo samples use canlah.ai or fictional brands.
- New project, started on 2026-11-05 during the Kiroween submission period. No pre-existing code is incorporated; only published open-source libraries listed in `package.json`.

## License
[MIT](./LICENSE)
