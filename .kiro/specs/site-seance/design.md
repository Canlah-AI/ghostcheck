# Design: Site Séance

## Overview

Site Séance is a deterministic, key-free analysis pipeline. The user pastes a public URL into the React frontend; the frontend routes all HTTP fetches through a Cloudflare Worker that enforces SSRF guards; the raw text responses are handed to `packages/core`, a pure TypeScript package that runs every check and computes a Haunting Score. No state is stored between runs—the same URL always produces the same result.

The pipeline has four layers:

```
Browser (apps/web)  →  Fetch Proxy (apps/worker)  →  public internet
                                    ↓ raw text
                        Core checks (packages/core)
                                    ↓ typed results
                        Browser renders score + evidence
```

---

## Architecture

### Sequence

1. User submits a URL in the web app.
2. Web app validates the scheme (`http` / `https`) before sending anything to the worker.
3. Web app calls the Worker once for each resource: homepage HTML, `/robots.txt`, `/llms.txt`, `/sitemap.xml`.
4. Worker validates the target (scheme, IP range), fetches with GET, enforces the 8 s timeout, 2 MB cap, and 3-redirect limit, then returns `{ status, body, finalUrl }`.
5. Web app collects all four responses and passes them to `packages/core` in-process.
6. Core runs each check function synchronously (pure functions, no I/O) and returns `CheckResult[]`.
7. Core's `scoreChecks` function folds the results into a `HauntingScore` (0–100).
8. Web app renders results; every evidence anchor points to the raw fetched text.

### Packages

| Package | Role |
|---|---|
| `packages/core` | Pure TypeScript checks + scorer. No network, no DOM. |
| `apps/worker` | Cloudflare Worker. Fetches URLs, enforces SSRF rules, returns raw text. |
| `apps/web` | Vite + React SPA. UI, URL input, result display. |

---

## Components and Interfaces

### `apps/worker` — Fetch Proxy

```typescript
// POST /fetch   body: { url: string }
// Response: FetchProxyResponse
interface FetchProxyResponse {
  ok: boolean;          // false when blocked or errored
  status: number | null;
  body: string | null;  // raw text, null on timeout/blocked
  finalUrl: string;     // URL after redirects
  error?: string;       // human-readable reason when ok=false
}
```

The worker performs one SSRF check on the initial URL, then re-checks the resolved IP after each redirect. Private ranges to reject: RFC 1918, loopback (127.0.0.0/8, ::1), link-local (169.254.0.0/16, fe80::/10), CGNAT (100.64.0.0/10), cloud metadata (169.254.169.254, fd00:ec2::254).

### `packages/core` — Check Functions

Each check lives in one file under `packages/core/src/checks/` and exports:

```typescript
export interface CheckResult {
  id: string;           // stable identifier, e.g. "robots-gptbot"
  label: string;        // short human label
  passed: boolean;
  score: number;        // points earned: 0 or weight
  weight: number;       // max points this check contributes
  evidence: Evidence[];
}

export interface Evidence {
  label: string;        // e.g. "robots.txt line 12"
  excerpt: string;      // raw text slice shown in UI
  sourceUrl: string;    // URL the text came from
}

// Every check function signature
export type CheckFn = (inputs: SiteInputs) => CheckResult;

export const weight: number; // exported alongside the function
```

`SiteInputs` bundles everything a check might read:

```typescript
export interface SiteInputs {
  url: string;          // normalized, scheme + host
  homepageHtml: FetchResult;
  robotsTxt: FetchResult;
  llmsTxt: FetchResult;
  sitemapXml: FetchResult;
}

export interface FetchResult {
  status: number | null;  // null = fetch failed / timed out
  body: string | null;
  finalUrl: string;
}
```

### `packages/core` — Scoring

```typescript
export interface HauntingScore {
  total: number;         // 0–100, integer
  maxPossible: number;   // sum of all weights (always ≤ 100)
  breakdown: CheckResult[];
}

export function scoreChecks(results: CheckResult[]): HauntingScore;
```

`scoreChecks` sums `result.score` for all results, clamps to `[0, maxPossible]`, then scales linearly to `[0, 100]`.

### Individual Check Modules

| File | Check ID prefix | Requirement |
|---|---|---|
| `checks/robots.ts` | `robots-*` | 2 |
| `checks/llmstxt.ts` | `llmstxt-*` | 3 |
| `checks/jsonld.ts` | `jsonld-*` | 4 |
| `checks/visibleText.ts` | `visible-text` | 5 |
| `checks/meta.ts` | `meta-*` | 6 |

### `packages/core` — Parsers

```typescript
// robots.ts
export function parseRobotsTxt(raw: string): RobotsTxtDocument;
export function isAllowed(doc: RobotsTxtDocument, userAgent: string, path: string): RobotsVerdict;

interface RobotsVerdict {
  allowed: boolean;
  matchedRule: string | null;   // exact line from raw text
  matchedGroup: string;         // user-agent that matched
}

// llmstxt.ts
export function parseLlmsTxt(raw: string): LlmsTxtReport;
interface LlmsTxtReport {
  hasH1: boolean;
  hasOptionalSummary: boolean;
  sections: LlmsTxtSection[];
  problems: string[];
}

// jsonld.ts
export function extractJsonLd(html: string): JsonLdReport;
interface JsonLdReport {
  blocks: Array<{ raw: string; parsed: unknown | null; types: string[] }>;
  parseErrors: string[];
  hasOrganization: boolean;
  hasLocalBusiness: boolean;
  hasProduct: boolean;
  hasFaqPage: boolean;
}

// visibleText.ts
export function extractVisibleText(html: string): VisibleTextReport;
interface VisibleTextReport {
  wordCount: number;
  excerpt: string;  // first 500 chars for evidence display
}

// meta.ts
export function extractMeta(html: string): MetaReport;
interface MetaReport {
  title: string | null;
  description: string | null;
  canonical: string | null;
  hreflang: Array<{ lang: string; href: string }>;
}
```

### `apps/web` — React Components

| Component | Responsibility |
|---|---|
| `URLInputForm` | Validates scheme; submits to worker; shows loading state |
| `SiteSeanceReport` | Renders `HauntingScore` + `CheckResult[]` |
| `ScoreGauge` | Visual 0–100 gauge; ghost visibility increases with score |
| `CheckCard` | Single check with pass/fail badge, weight, evidence list |
| `EvidenceDrawer` | Shows raw fetched text anchored to a source URL |

---

## Data Models

```typescript
// Shared across core and web — published from packages/core/src/types.ts

export interface SiteInputs { ... }      // see above
export interface FetchResult { ... }     // see above
export interface CheckResult { ... }     // see above
export interface Evidence { ... }        // see above
export interface HauntingScore { ... }   // see above

// AI crawler registry (static, in packages/core/src/crawlers.ts)
export interface AiCrawler {
  userAgent: string;   // token to match in robots.txt
  name: string;        // display name, e.g. "GPTBot"
  docUrl: string;      // link to operator's public documentation
}

// Weight registry (static, in packages/core/src/weights.ts)
export const WEIGHTS: Record<string, number>;
// Total of all values in WEIGHTS must equal 100.
```

---

## Correctness Properties

These properties are suitable for fast-check property-based tests. Each maps to one or more requirements.

### P1 — Score bounds (_Requirements: 7.1, 7.2_)
For any array of `CheckResult` values where each `result.score ∈ [0, result.weight]` and `sum(weights) ≤ 100`, `scoreChecks(results).total` is always in `[0, 100]`.

### P2 — Score monotonicity (_Requirements: 7.3_)
Given a baseline array of `CheckResult` values and a derived array where exactly one result is changed from `passed=false, score=0` to `passed=true, score=weight` (all other fields identical), `scoreChecks(derived).total >= scoreChecks(baseline).total`.

### P3 — Weights sum to ≤ 100 (_Requirements: 7.1_)
The compile-time sum of all values in `WEIGHTS` is exactly 100. A property test generates random subsets of checks and confirms partial sums never exceed 100.

### P4 — robots.txt longest-path wins (_Requirements: 2.2_)
For any two rules with overlapping paths where one path is strictly longer than the other, the longer path's rule is always applied regardless of declaration order in the file.

### P5 — robots.txt Allow wins on tie (_Requirements: 2.2_)
When an Allow and a Disallow rule match the same path with equal length, `isAllowed` returns `true`.

### P6 — robots.txt case-insensitive user-agent matching (_Requirements: 2.2_)
For any user-agent string `ua`, `isAllowed(doc, ua.toUpperCase(), path)` equals `isAllowed(doc, ua.toLowerCase(), path)`.

### P7 — robots.txt fallback to `*` (_Requirements: 2.3_)
When no group matches the specific user-agent, the result equals what the `*` group would decide for the same path; if no `*` group exists, the result is `allowed=true`.

### P8 — robots.txt round-trip evidence (_Requirements: 2.4_)
The `matchedRule` field in `RobotsVerdict` is always a substring of the raw robots.txt input.

### P9 — JSON-LD parse errors never throw (_Requirements: 4.2_)
For any arbitrary string passed as `html`, `extractJsonLd` returns a `JsonLdReport` without throwing; malformed `<script type="application/ld+json">` blocks appear in `parseErrors`.

### P10 — visible text excludes script/style/noscript/template (_Requirements: 5.1_)
For any HTML string that contains only `<script>`, `<style>`, `<noscript>`, and `<template>` elements with content, `extractVisibleText(html).wordCount === 0`.

### P11 — visible text word count is non-negative (_Requirements: 5.1_)
For any HTML string, `extractVisibleText(html).wordCount >= 0`.

### P12 — llms.txt problem list is empty for a valid document (_Requirements: 3.1_)
A string matching the canonical llmstxt.org format (H1, optional blockquote, one or more H2 sections with markdown links) always produces `problems.length === 0`.

### P13 — FetchResult with null body is handled gracefully (_Requirements: 1.5_)
For any check function `f`, calling `f({ ...inputs, robotsTxt: { status: null, body: null, finalUrl: '' } })` returns a `CheckResult` without throwing.

---

## Error Handling

| Failure | Handling |
|---|---|
| Non-http/https URL | Rejected in the web app before any network call; plain error message. |
| Worker: private/loopback IP resolved | `ok: false, error: "blocked: private address"`. |
| Worker: redirect to private IP | Re-checked after each redirect; same rejection. |
| Worker: timeout (> 8 s) | `ok: false, error: "timeout"`. Body is null. |
| Worker: body > 2 MB | Response truncated at 2 MB; header `X-Truncated: true` set. |
| Worker: > 3 redirects | `ok: false, error: "too many redirects"`. |
| Resource HTTP 404 | `FetchResult { status: 404, body: null }`. Check records "not found" evidence. |
| JSON-LD parse failure | Error recorded in `JsonLdReport.parseErrors`; other blocks continue. |
| Entirely missing resource (llms.txt, sitemap.xml) | Check marks itself as failed and provides a "missing" evidence item. |
| Unexpected exception in a check | The check catches internally, returns `passed: false, score: 0, evidence: [{ label: "error", excerpt: message }]`. |

All errors are surfaced as structured data in `CheckResult.evidence`. The UI never shows a blank screen; every error state has a user-visible explanation.

---

## Testing Strategy

### Unit tests (`*.test.ts`) — co-located with source

- Each parser (`parseRobotsTxt`, `parseLlmsTxt`, `extractJsonLd`, `extractVisibleText`, `extractMeta`) has example-based tests covering normal input, empty input, and known edge cases (UTF-8, unusual whitespace, missing sections).
- Each check function has tests for the "resource missing" path and the "resource present" path.
- `scoreChecks` has tests for all-pass, all-fail, partial, and boundary (single check) cases.
- Worker SSRF guard has tests for accepted public IPs, rejected private ranges (IPv4 and IPv6), redirect re-check, timeout simulation, and body truncation.

### Property-based tests (`*.pbt.test.ts`) — co-located with source

Each of P1–P13 above becomes one `fc.property` test using fast-check arbitraries:

- `fc.array(fc.record({ score, weight, passed }))` for score properties (P1–P3).
- `fc.string()` / `fc.array(fc.string())` for parser properties (P4–P13).
- Robots.txt properties (P4–P8) use a custom `fc.Arbitrary<RobotsTxtInput>` that generates valid rule sets.

### Integration smoke test

A single Vitest test that assembles a `SiteInputs` object from fixture files under `packages/core/src/__fixtures__/` (static text, no network) and asserts that `scoreChecks(runAllChecks(inputs)).total` is in `[0, 100]`. This is the only test that crosses check boundaries.

### Worker tests

The Worker is tested with Vitest using `miniflare` or the `@cloudflare/workers-types` compatible test harness. Tests confirm SSRF rejection, timeout, redirect limit, and correct passthrough of a public URL response.

### What is NOT tested here

AI Séance (covered by its own spec), the React component tree (covered by the haunted-ui spec), and real network calls (the worker integration test uses mocked fetch).
