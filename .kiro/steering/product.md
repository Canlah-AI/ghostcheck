---
inclusion: always
---
# Product: GhostCheck

**One line:** Is your brand a ghost to AI assistants? Paste a URL and see how visible your site is to AI crawlers and AI answers.

## Users
- Owners and marketers of small and mid-sized businesses who want to know why AI assistants do not mention them.
- Hackathon judges, who must be able to use the full app **without an API key and without signing in**.

## Features (V1)
1. **Site Séance** (no key, deterministic): fetch a public URL and check robots.txt rules for AI crawlers, /llms.txt, schema.org JSON-LD, visible text in raw HTML, and basic meta tags. Produce a 0–100 **Haunting Score** with public weights. Every point links to the raw evidence.
2. **AI Séance** (optional, bring your own key): generate 5–10 editable buying-intent questions from brand + category + city, ask 1–2 AI model APIs with the user's own key, and report whether the brand is mentioned, in what position, and which other names appear. A **demo mode** replays a static sample result committed to the repo.
3. **Haunted UI** (Costume Contest): a haunted-house interface where the ghost becomes more visible as the score rises.

## Non-goals (V1)
Accounts, databases, paywalls, scheduled monitoring, multiple languages, browser extensions.

## Honesty rules (product copy)
- Never claim a ranking we cannot measure (no "you rank #3 in an assistant").
- Always say: an API answer is not the same as what a person sees in a consumer chat app.
- Every number shown must come from a check the user can open and inspect.
- No third-party logos or trademarks in the UI or demo video. Use plain text labels such as "Assistant A" plus the API model id.

## Theme
Kiroween 2026, Costume Contest category. Spooky but readable: WCAG AA contrast, respects `prefers-reduced-motion`.
