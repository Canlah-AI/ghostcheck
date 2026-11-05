---
inclusion: always
---
# Project structure

```
/.kiro/            Kiro steering, specs and hooks (must stay at repo root, never gitignored)
/packages/core/    pure TS checks + scoring (no network, no DOM)
/apps/web/         Vite + React haunted UI
/apps/worker/      Cloudflare Worker fetch proxy
/demo/             static sample results for demo mode (fictional or canlah.ai only)
/docs/             KIRO_USAGE.md and screenshots
```

- One check = one file in `packages/core/src/checks/`, exporting a function and its weight.
- Tests sit next to code as `*.test.ts`; property tests as `*.pbt.test.ts`.
