# Tasks: Site Séance

## Implementation Plan

- [ ] 1. Bootstrap the npm workspace and package skeletons
  - [ ] 1.1 Create root `package.json` (`workspaces: ["packages/*","apps/*"]`, `"private": true`) plus `packages/core/package.json` (`@ghostcheck/core`, `"type":"module"`, `vitest`, `fast-check`, strict TypeScript), `apps/worker/package.json` (`wrangler`), and `apps/web/package.json` (`vite`, `react`, `react-dom`, workspace ref to `@ghostcheck/core`).
  - [ ] 1.2 Add root `tsconfig.json` in strict mode with project references. Add `.gitignore` entries for `dist/`, `node_modules/`, `.wrangler/`, `.dev.vars`.

  _Requirements: 1.1_

- [ ] 2. Define shared TypeScript types in `packages/core`
  - [ ] 2.1 Create `packages/core/src/types.ts` exporting `FetchResult`, `SiteInputs`, `Evidence`, `CheckResult`, `HauntingScore`, `CheckFn`, `AiCrawler`.
  - [ ] 2.2 Create `packages/core/src/crawlers.ts` exporting `AI_CRAWLERS: AiCrawler[]` — every entry must have a valid operator `docUrl`.
  - [ ] 2.3 Create `packages/core/src/weights.ts` exporting `WEIGHTS: Record<string, number>` where all values sum to exactly 100.

  _Requirements: 2.1, 7.1_

- [ ] 3. Implement the robots.txt parser and matcher
  - [ ] 3.1 Create `packages/core/src/parsers/robots.ts`: `parseRobotsTxt` and `isAllowed` following RFC 9309 (case-insensitive UA, longest-path wins, Allow wins tie, `*` fallback, missing group = allowed).
  - [ ] 3.2 Write unit tests (`robots.test.ts`) covering normal rules, empty file, comments-only, UTF-8 paths.
  - [ ] 3.3 Write property-based tests (`robots.pbt.test.ts`) for P4 (longest-path), P5 (Allow wins tie), P6 (case-insensitive UA), P7 (* fallback), P8 (matchedRule is substring of raw).

  _Requirements: 2.2, 2.3, 2.4_

- [ ] 4. Implement the llms.txt, JSON-LD, visible-text, and meta parsers
  - [ ] 4.1 `parsers/llmstxt.ts`: check H1, optional blockquote, H2 sections with markdown links; populate `problems`. Unit tests: valid doc, missing H1, empty input. PBT P12 (valid doc → zero problems).
  - [ ] 4.2 `parsers/jsonld.ts`: extract all `<script type="application/ld+json">` blocks (incl. `@graph`); detect `Organization`, `LocalBusiness`, `Product`, `FAQPage`; record parse errors without throwing. Unit tests: valid, `@graph`, malformed, no blocks. PBT P9 (never throws for any HTML).
  - [ ] 4.3 `parsers/visibleText.ts`: strip `<script>`, `<style>`, `<noscript>`, `<template>`; count words; populate `excerpt` (first 500 chars). Unit tests: normal, script-only, empty, nested. PBT P10 (script-only → wordCount 0), P11 (wordCount ≥ 0).
  - [ ] 4.4 `parsers/meta.ts`: read `<title>`, `<meta name="description">`, `<link rel="canonical">`, `<link rel="alternate" hreflang>`. Unit tests: all fields present, each missing individually, multiple hreflang, malformed tags.

  _Requirements: 3.1, 3.2, 4.1, 4.2, 4.3, 5.1, 6.1_

- [ ] 5. Implement the five check modules
  - [ ] 5.1 `checks/robots.ts`: per `AI_CRAWLERS` entry call `isAllowed` for `/`; one `CheckResult` per crawler with matched rule as evidence; export `weight`.
  - [ ] 5.2 `checks/llmstxt.ts`, `checks/jsonld.ts`, `checks/visibleText.ts`, `checks/meta.ts`: each passes or fails per its requirements; evidence from parsed data; export `weight`.
  - [ ] 5.3 Write unit tests for each check module: resource missing (null body), present + passing, present + failing.

  _Requirements: 2.4, 3.1, 3.2, 4.1, 4.3, 5.1, 5.2, 6.1_

- [ ] 6. Implement the scorer
  - [ ] 6.1 `packages/core/src/scorer.ts`: `scoreChecks(results)` sums `result.score`, clamps to `[0, maxPossible]`, scales to `[0, 100]`.
  - [ ] 6.2 `packages/core/src/index.ts`: `runAllChecks(inputs)` calls all five check modules in sequence.
  - [ ] 6.3 Unit tests: all-pass, all-fail, partial, zero-weight edge cases. PBT P1 (bounded 0–100), P2 (monotonic), P3 (weights ≤ 100).

  _Requirements: 7.1, 7.2, 7.3, 7.4_

- [ ] 7. Implement the Cloudflare Worker fetch proxy
  - [ ] 7.1 `apps/worker/src/index.ts`: accept `POST /fetch` with `{ url: string }`; reject non-http/https before any DNS lookup; implement SSRF IP guard (RFC 1918, loopback, link-local, CGNAT, cloud metadata), re-checked after each redirect; GET only, 8 s timeout, 2 MB cap (`X-Truncated: true`), max 3 redirects.
  - [ ] 7.2 `apps/worker/wrangler.toml`: worker name `ghostcheck-proxy`, compatibility date, no secrets or bindings.
  - [ ] 7.3 Unit tests (`index.test.ts`): valid public URL, private IPv4/IPv6, loopback, link-local, CGNAT, cloud metadata, timeout, body truncation, redirect limit, non-http scheme.

  _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

- [ ] 8. Add core integration smoke test
  - [ ] 8.1 Create fixture files under `packages/core/src/__fixtures__/` (`homepage.html`, `robots.txt`, `llms.txt`, `sitemap.xml`) using fictional or canlah.ai content — no real customer data.
  - [ ] 8.2 `integration.test.ts`: load fixtures → `runAllChecks` → `scoreChecks`; assert `total ∈ [0, 100]` and every `CheckResult` has non-empty `id` and `label`.

  _Requirements: 1.5, 7.1, 7.2_

- [ ] 9. Build the React web app scaffold
  - [ ] 9.1 Initialise `apps/web` with Vite React-TypeScript template; configure path alias for `@ghostcheck/core`; create `App.tsx` top-level routing: `URLInputForm` → loading → `SiteSeanceReport`.
  - [ ] 9.2 `URLInputForm.tsx`: validate scheme, call Worker for four resources in parallel, feed results to `runAllChecks` + `scoreChecks`.
  - [ ] 9.3 `SiteSeanceReport.tsx` + `ScoreGauge.tsx` + `CheckCard.tsx` + `EvidenceDrawer.tsx`: render score, checks, and raw evidence; `ScoreGauge` ghost opacity tracks score and respects `prefers-reduced-motion`.

  _Requirements: 1.1, 2.4, 3.2, 4.3, 5.2, 6.1, 7.4_

- [ ] 10. Apply Haunted-UI theme and accessibility
  - [ ] 10.1 `src/styles/theme.css`: CSS custom properties for spooky palette; all foreground/background pairs ≥ 4.5:1 contrast (text) and ≥ 3:1 (UI components); `@media (prefers-reduced-motion: reduce)` blocks disable all animations.
  - [ ] 10.2 All interactive elements have visible focus indicators, all images have alt text, all form inputs have labels.
  - [ ] 10.3 Run `vitest-axe` (or `@axe-core/react`) locally against `SiteSeancePage` with a mock fixture; assert zero violations.

  _Requirements: 7.4_

- [ ] 11. Wire Worker URL, configure deployment, and verify
  - [ ] 11.1 `VITE_WORKER_URL` env var read from `.env.local` (never committed). `apps/web/public/_headers` (CSP, `X-Content-Type-Options`, `X-Frame-Options`). `apps/web/public/_redirects` (SPA routing).
  - [ ] 11.2 Document manual two-step deploy (Worker: `wrangler deploy`; Pages: `wrangler pages deploy`) in `docs/KIRO_USAGE.md` — no secrets in doc.
  - [ ] 11.3 Confirm no-secrets compliance is enforced by the `secret-scan` Kiro hook (`.kiro/hooks/secret-scan.kiro.hook`) — no separate CI step required.

  _Requirements: 1.1, 1.2, 1.3, 1.4_
