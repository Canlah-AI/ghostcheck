---
inclusion: always
---
# Security and data rules (hard rules)

1. **No secrets in the repo or the deployed app.** No API keys, tokens or passwords in code, config, tests, fixtures, screenshots or docs. Anything matching `sk-`, `AIza`, `ghp_`, `gho_`, `xox`, `AKIA` or a PEM header must never be committed.
2. **Bring your own key (BYOK).** AI Séance keys live only in browser memory for the current tab. Never send them to our Worker, never write them to localStorage, cookies or logs.
3. **SSRF protection in the Worker.** Only `http`/`https`. Resolve and reject private, loopback, link-local, CGNAT and cloud metadata ranges (IPv4 and IPv6), including after redirects. GET only, 8 s timeout, 2 MB response cap, at most 3 redirects.
4. **No customer data.** Fixtures and demo results use fictional brands or canlah.ai only. No real audit results, no client names.
5. **Built from scratch.** This project does not copy or port code from any other Canlah product. Do not add code from outside this repo except published open-source libraries declared in package.json.
