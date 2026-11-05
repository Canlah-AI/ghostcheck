# Tasks: Site Séance

## Implementation Plan

- [ ] 1. Bootstrap the npm workspace and package skeletons
  - [ ] 1.1 Create root `package.json` with `workspaces: ["packages/*", "apps/*"]` and `"private": true`.
  - [ ] 1.2 Create `packages/core/package.json` with name `@ghostcheck/core`, `"type": "module"`, `vitest`, `fast-check`, and TypeScript dev-dependencies (strict tsconfig).
  - [ ] 1.3 Create `apps/worker/package.json` with name `@ghostcheck/worker` and `wrangler` as a dev-dependency.
  - [ ] 1.4 Create `apps/web/package.json` with name `@ghostcheck/web`, `vite`, `react`, `react-dom`, and `@ghostcheck/core` workspace reference.
  - [ ] 1.5 Add root `tsconfig.json` in strict mode with project references for each package/app.
  - [ ] 1.6 Add `.gitignore` entries for `dist/`, `node_modules/`, `.wrangler/`, `.dev.vars`.

  _Requirements: 1.1_

- [ ] 2. Define shared TypeScript types in `packages/core`
  - [ ] 2.1 Create `packages/core/src/types.ts` exporting `FetchResult`, `SiteInputs`, `Evidence`, `CheckResult`, `HauntingScore`, `CheckFn`, `AiCrawler`.
  - [ ] 2.2 Create `packages/core/src/crawlers.ts` exporting `AI_CRAWLERS: AiCrawler[]` — a static list of AI crawler user-agent tokens, display names, and their operator documentation URLs. Every entry must have a valid `docUrl`.
  - [ ] 2.3 Create `packages/core/src/weights.ts` exporting `WEIGHTS: Record<string, number>` where all values sum to exactly 100.

  _Requirements: 2.1, 7.1_

- [ ] 3. Implement the robots.txt parser and matcher
  - [ ] 3.1 Create `packages/core/src/parsers/robots.ts` implementing `parseRobotsTxt(raw: string): RobotsTxtDocument` that handles multiple user-agent groups, `Allow` and `Disallow` directives, blank lines, and comments.
  - [ ] 3.2 Implement `isAllowed(doc: RobotsTxtDocument, userAgent: string, path: string): RobotsVerdict` following RFC 9309: case-insensitive user-agent matching, longest-path wins, Allow wins on a tie, `*` fallback, missing group treated as allowed.
  - [ ] 3.3 Write example-based unit tests in `packages/core/src/parsers/robots.test.ts` covering normal rules, empty file, comments-only file, UTF-8 paths.
  - [ ] 3.4 Write property-based tests in `packages/core/src/parsers/robots.pbt.test.ts` implementing properties P4, P5, P6, P7, and P8 with fast-check.

  _Requirements: 2.2, 2.3, 2.4_

- [ ] 4. Implement the llms.txt parser
  - [ ] 4.1 Create `packages/core/src/parsers/llmstxt.ts` implementing `parseLlmsTxt(raw: string): LlmsTxtReport` that checks for an H1 title, optional blockquote summary, and H2 sections with markdown links.
  - [ ] 4.2 Populate `problems` with a descriptive string for each structural violation (missing H1, H2 section with no links, etc.).
  - [ ] 4.3 Write unit tests in `packages/core/src/parsers/llmstxt.test.ts` covering a valid document, missing H1, missing sections, and empty input.
  - [ ] 4.4 Write property-based tests in `packages/core/src/parsers/llmstxt.pbt.test.ts` implementing property P12.

  _Requirements: 3.1, 3.2_

- [ ] 5. Implement the JSON-LD extractor
  - [ ] 5.1 Create `packages/core/src/parsers/jsonld.ts` implementing `extractJsonLd(html: string): JsonLdReport` that finds all `<script type="application/ld+json">` blocks, handles `@graph` arrays, and records parse errors without throwing.
  - [ ] 5.2 Detect presence of `Organization`, `LocalBusiness` (and subtypes via `@type` string includes), `Product`, and `FAQPage` types.
  - [ ] 5.3 Write unit tests in `packages/core/src/parsers/jsonld.test.ts` covering valid JSON-LD, `@graph`, malformed JSON, nested `@type`, and no blocks present.
  - [ ] 5.4 Write property-based tests in `packages/core/src/parsers/jsonld.pbt.test.ts` implementing property P9.

  _Requirements: 4.1, 4.2, 4.3_

- [ ] 6. Implement the visible-text extractor
  - [ ] 6.1 Create `packages/core/src/parsers/visibleText.ts` implementing `extractVisibleText(html: string): VisibleTextReport` that strips `<script>`, `<style>`, `<noscript>`, and `<template>` elements and counts words in the remaining text nodes.
  - [ ] 6.2 Populate `excerpt` with the first 500 characters of extracted text for use as evidence.
  - [ ] 6.3 Write unit tests in `packages/core/src/parsers/visibleText.test.ts` covering a normal page, a page with only scripts/styles, empty HTML, and deeply nested content.
  - [ ] 6.4 Write property-based tests in `packages/core/src/parsers/visibleText.pbt.test.ts` implementing properties P10 and P11.

  _Requirements: 5.1_

- [ ] 7. Implement the meta extractor
  - [ ] 7.1 Create `packages/core/src/parsers/meta.ts` implementing `extractMeta(html: string): MetaReport` that reads `<title>`, `<meta name="description">`, `<link rel="canonical">`, and `<link rel="alternate" hreflang="...">` tags.
  - [ ] 7.2 Write unit tests in `packages/core/src/parsers/meta.test.ts` covering all four fields present, each field individually missing, multiple hreflang entries, and malformed tags.

  _Requirements: 6.1_

- [ ] 8. Implement the five check modules
  - [ ] 8.1 Create `packages/core/src/checks/robots.ts`: for each entry in `AI_CRAWLERS`, call `isAllowed` for `/`, build one `CheckResult` per crawler with the matched rule line as evidence, and export `weight`.
  - [ ] 8.2 Create `packages/core/src/checks/llmstxt.ts`: return pass if `/llms.txt` exists and `parseLlmsTxt` reports zero problems; return fail with evidence listing problems or "missing". Export `weight`.
  - [ ] 8.3 Create `packages/core/src/checks/jsonld.ts`: return pass if `extractJsonLd` finds at least one qualifying type (`Organization`, `LocalBusiness`, `Product`, or `FAQPage`). Evidence includes block count and found types. Export `weight`.
  - [ ] 8.4 Create `packages/core/src/checks/visibleText.ts`: return pass if `wordCount > 0`; evidence shows word count and excerpt. Export `weight`.
  - [ ] 8.5 Create `packages/core/src/checks/meta.ts`: one `CheckResult` per meta field (title, description, canonical, hreflang); each passes if the field is present and non-empty. Export `weight`.
  - [ ] 8.6 Write unit tests for each check module covering: resource missing (null body), resource present + passing, resource present + failing.

  _Requirements: 2.4, 3.1, 3.2, 4.1, 4.3, 5.1, 5.2, 6.1_

- [ ] 9. Implement the scorer
  - [ ] 9.1 Create `packages/core/src/scorer.ts` implementing `scoreChecks(results: CheckResult[]): HauntingScore` that sums `result.score`, clamps to `[0, maxPossible]`, and scales to `[0, 100]`.
  - [ ] 9.2 Create `packages/core/src/index.ts` exporting `runAllChecks(inputs: SiteInputs): CheckResult[]` that calls all five check modules in sequence and returns the flat array.
  - [ ] 9.3 Write unit tests in `packages/core/src/scorer.test.ts` covering all-pass, all-fail, partial pass, and zero-weight edge cases.
  - [ ] 9.4 Write property-based tests in `packages/core/src/scorer.pbt.test.ts` implementing properties P1, P2, and P3.

  _Requirements: 7.1, 7.2, 7.3, 7.4_

- [ ] 10. Implement the Cloudflare Worker fetch proxy
  - [ ] 10.1 Create `apps/worker/src/index.ts` as a Cloudflare Worker that accepts `POST /fetch` with body `{ url: string }` and returns `FetchProxyResponse`.
  - [ ] 10.2 Implement URL validation: reject non-http/https schemes with `ok: false` before any DNS lookup.
  - [ ] 10.3 Implement SSRF IP guard: resolve the hostname and reject RFC 1918, loopback, link-local (169.254.x.x, fe80::/10), CGNAT (100.64.0.0/10), and cloud metadata (169.254.169.254) ranges. Re-check after each redirect.
  - [ ] 10.4 Enforce GET only, 8 s timeout, 2 MB body cap (truncate with `X-Truncated: true` header), and at most 3 redirects.
  - [ ] 10.5 Return `{ ok: true, status, body, finalUrl }` on success; `{ ok: false, error, status: null, body: null, finalUrl: originalUrl }` on any rejection or error.
  - [ ] 10.6 Create `apps/worker/wrangler.toml` with worker name `ghostcheck-proxy`, compatibility date, and no secrets or bindings.
  - [ ] 10.7 Write unit tests in `apps/worker/src/index.test.ts` covering: valid public URL, private IPv4, private IPv6, loopback, link-local, CGNAT, cloud metadata, timeout, body truncation, redirect limit, non-http scheme.

  _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

- [ ] 11. Add core integration smoke test
  - [ ] 11.1 Create fixture files under `packages/core/src/__fixtures__/`: `homepage.html`, `robots.txt`, `llms.txt`, `sitemap.xml` — all fictional or canlah.ai content, no real customer data.
  - [ ] 11.2 Write `packages/core/src/integration.test.ts` that loads the fixtures, assembles `SiteInputs`, calls `runAllChecks`, then `scoreChecks`, and asserts `total` is in `[0, 100]` and each `CheckResult` has non-empty `id` and `label`.

  _Requirements: 1.5, 7.1, 7.2_

- [ ] 12. Build the React web app scaffold
  - [ ] 12.1 Initialise `apps/web` with Vite's React-TypeScript template; configure `vite.config.ts` to resolve `@ghostcheck/core` from the workspace.
  - [ ] 12.2 Create `apps/web/src/App.tsx` with top-level routing: `URLInputForm` → loading state → `SiteSeanceReport`.
  - [ ] 12.3 Create `apps/web/src/components/URLInputForm.tsx`: validates scheme client-side, calls the Worker for each of the four resources in parallel, collects `FetchResult` objects, then calls `runAllChecks` + `scoreChecks` from core.
  - [ ] 12.4 Create `apps/web/src/components/SiteSeanceReport.tsx`: renders `ScoreGauge` and a list of `CheckCard` components from `HauntingScore.breakdown`.
  - [ ] 12.5 Create `apps/web/src/components/ScoreGauge.tsx`: a numeric 0–100 display; the ghost SVG opacity or visibility increases linearly with score. Respects `prefers-reduced-motion`.
  - [ ] 12.6 Create `apps/web/src/components/CheckCard.tsx`: shows check label, pass/fail badge, weight, earned points, and an expandable evidence list.
  - [ ] 12.7 Create `apps/web/src/components/EvidenceDrawer.tsx`: renders the raw text excerpt and a linked source URL for each `Evidence` item.

  _Requirements: 1.1, 2.4, 3.2, 4.3, 5.2, 6.1, 7.4_

- [ ] 13. Apply Haunted-UI theme and accessibility
  - [ ] 13.1 Create `apps/web/src/styles/theme.css` with CSS custom properties for the spooky palette; all foreground/background pairs must meet WCAG AA contrast (≥ 4.5:1 for text, ≥ 3:1 for UI components).
  - [ ] 13.2 Add `@media (prefers-reduced-motion: reduce)` blocks to disable or replace all CSS animations and transitions.
  - [ ] 13.3 Ensure all interactive elements have visible focus indicators, all images have alt text, and all form inputs have associated labels.

  _Requirements: 7.4_

- [ ] 14. Wire the Worker URL into the web app and configure deployment
  - [ ] 14.1 Add a `VITE_WORKER_URL` environment variable (read from `.env.local`, never committed); the web app reads it at build time.
  - [ ] 14.2 Create `apps/web/public/_headers` to set `Content-Security-Policy`, `X-Content-Type-Options`, and `X-Frame-Options`.
  - [ ] 14.3 Add `apps/web/public/_redirects` for Cloudflare Pages SPA routing (`/* /index.html 200`).
  - [ ] 14.4 Document the two-step deploy process (Worker first, then Pages) in `docs/KIRO_USAGE.md` with no secrets or API keys in the doc.

  _Requirements: 1.1, 1.2, 1.3, 1.4_
