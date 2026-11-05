# Design: Haunted UI

## Overview

The Haunted UI is the visual shell of GhostCheck. It frames every feature inside a haunted-house
metaphor: the brand being checked is a ghost whose opacity and form track the Haunting Score
produced by Site Séance. Low score → faint, barely-there ghost. High score → solid, visible
ghost. Completing each check animates the ghost gaining a new feature (eyes, outline, …).

The UI is a static single-page application built with Vite + React (TypeScript strict mode),
hosted on Cloudflare Pages. It has no back-end of its own beyond the fetch proxy defined in
`apps/worker`. All state is in-memory; nothing is written to `localStorage`, cookies or the
network except the proxied site fetches and (in AI Séance live mode) direct browser-to-provider
calls.

Two primary journeys:

1. **Judge path** — arrive, enter or accept a sample URL, see Site Séance results and ghost
   animation within two clicks, no sign-in, no key.
2. **Power-user path** — enter brand details, supply an API key or launch demo mode, view AI
   Séance results.

Accessibility is non-negotiable: WCAG 2.1 AA contrast throughout, full keyboard navigation,
screen-reader announcements for dynamic updates, and a `prefers-reduced-motion` mode that turns
off all animation.

---

## Architecture

```
apps/web/src/
│
├── main.tsx                    # Vite entry; mounts <App />
├── App.tsx                     # Route shell; manages top-level AppState
│
├── haunted-ui/
│   ├── GhostCanvas.tsx         # SVG/CSS ghost; reads ghostLevel (0–4) prop
│   ├── FogLayer.tsx            # CSS-animated fog; disabled on reduced-motion
│   ├── HauntingScore.tsx       # Numeric score display + progress ring
│   ├── CheckEvidence.tsx       # Single check result row with evidence link
│   ├── CheckList.tsx           # List of CheckEvidence rows
│   ├── JudgePath.tsx           # Two-click entry point with sample URL
│   └── ThemeProvider.tsx       # CSS custom-property theming; spooky palette
│
├── site-seance/
│   └── SiteSeancePage.tsx      # URL input + trigger; renders haunted-ui results
│
├── ai-seance/
│   └── AiSeancePage.tsx        # Key entry / demo mode; renders AI results
│
└── shared/
    ├── ErrorBoundary.tsx
    ├── KeyboardTrap.tsx        # Focus management for modal regions
    └── SkipLink.tsx            # "Skip to main content" accessibility link
```

The `packages/core` library is a pure-TS dependency. The UI imports its types and pure functions
but owns all rendering, state management, and side effects.

```
┌──────────────────────────────────────────────────────────────────┐
│  Browser tab (apps/web)                                          │
│                                                                  │
│  App.tsx ──▶ SiteSeancePage ──▶ GhostCanvas (score → level)     │
│           │                 └─▶ CheckList ──▶ CheckEvidence      │
│           │                 └─▶ HauntingScore                    │
│           │                                                      │
│           └─▶ AiSeancePage  (existing AI Séance design)          │
│                                                                  │
│  JudgePath (pre-fills sample URL; opens demo mode)               │
│                                                                  │
│  FogLayer (CSS animation; off when prefers-reduced-motion)       │
│  ThemeProvider (spooky CSS vars; WCAG AA palette)                │
└──────────────────────────────────────────────────────────────────┘
         │
         │  HTTPS GET (public URLs only)
         ▼
   apps/worker  (Cloudflare Worker fetch proxy)
         │
         ▼
   packages/core  (pure TS; no network; no DOM)
   ├── scoring/   (HauntingScore, scoreChecks)
   └── checks/    (robots, llms-txt, schema, meta, text)
```

---

## Components and Interfaces

### `GhostCanvas`

Renders an SVG ghost whose visual completeness is driven by a discrete `ghostLevel` value
(0–4) derived from the Haunting Score. Each level adds one SVG layer (body outline, eyes,
smile, glow, full form). Transitions between levels use a CSS opacity/scale animation that is
suppressed under `prefers-reduced-motion`.

```ts
export interface GhostCanvasProps {
  /** 0 (invisible) → 4 (fully materialised). Derived from score by caller. */
  ghostLevel: 0 | 1 | 2 | 3 | 4;
  /** Accessible label announced when level changes, e.g. "Ghost level 3 of 4". */
  ariaLabel: string;
}
```

Level mapping (informative, set by caller):

| Score range | ghostLevel |
|-------------|-----------|
| 0–19        | 0          |
| 20–39       | 1          |
| 40–59       | 2          |
| 60–79       | 3          |
| 80–100      | 4          |

### `FogLayer`

CSS-only animated fog overlay, absolutely positioned behind content. Does not render at all when
`window.matchMedia('(prefers-reduced-motion: reduce)').matches` is `true` at mount time and
stays inactive if the media query fires later.

```ts
export interface FogLayerProps {
  /** Allow parent to force-disable regardless of system preference. */
  disabled?: boolean;
}
```

### `HauntingScore`

Displays the numeric score (0–100) inside an SVG progress ring. Updates via a count-up animation
(suppressed under reduced-motion: value jumps directly). Uses `aria-live="polite"` to announce
the final value to screen readers.

```ts
export interface HauntingScoreProps {
  score: number;          // 0–100, integer
  /** Shown below the ring: e.g. "Your brand is barely a whisper." */
  flavourText: string;
}
```

### `CheckEvidence`

One row for a single check result. Contains the check name, pass/fail indicator, a short
summary, and a link to the raw evidence (fetched text or rule line).

```ts
export interface CheckEvidenceProps {
  name: string;
  passed: boolean;
  summary: string;
  /** URL fragment or external link to the raw evidence. */
  evidenceHref: string;
  /** Weight of this check in the Haunting Score, shown as tooltip. */
  weight: number;
}
```

The row's pass/fail indicator conveys state with both colour and icon (not colour alone) to
satisfy WCAG 1.4.1 (Use of Colour).

### `CheckList`

Renders an ordered list of `CheckEvidence` rows, grouped by check category (robots, llms.txt,
schema, meta, text). Each group heading is a `<h3>` within the page's heading hierarchy.
Keyboard focus moves naturally through the list; each evidence link is reachable via Tab.

```ts
export interface CheckListProps {
  checks: CheckResultSummary[];
  /** Called when a check transitions from fail→pass (drives ghost animation). */
  onCheckPass?: (checkName: string) => void;
}

export interface CheckResultSummary {
  name: string;
  passed: boolean;
  summary: string;
  evidenceHref: string;
  weight: number;
  category: "robots" | "llms-txt" | "schema" | "meta" | "text";
}
```

### `JudgePath`

Pre-fills the URL input with a sample URL (the `canlah.ai` fixture) so hackathon judges can
start Site Séance in one click. Provides a second button to open AI Séance demo mode. Total
click count from landing page to seeing results: ≤ 2.

```ts
export interface JudgePathProps {
  sampleUrl: string;         // pre-configured in constants, e.g. "https://canlah.ai"
  onRunSample: () => void;   // triggers Site Séance with sampleUrl
  onOpenDemo: () => void;    // opens AI Séance demo mode
}
```

### `ThemeProvider`

Wraps the app in a `<div>` that sets CSS custom properties for the spooky palette. All colours
must satisfy WCAG 2.1 AA (4.5:1 for normal text, 3:1 for large text and UI components) against
the background they appear on, including over the `FogLayer`.

```ts
export interface ThemeProviderProps {
  children: React.ReactNode;
}
```

Colour contract (CSS var names are illustrative):

| Variable              | Role                     | Contrast target         |
|-----------------------|--------------------------|-------------------------|
| `--color-bg`          | Page background          | —                       |
| `--color-surface`     | Card / panel surface     | —                       |
| `--color-text-primary`| Body text                | ≥ 4.5:1 on `--color-bg` |
| `--color-text-muted`  | Secondary text           | ≥ 4.5:1 on `--color-bg` |
| `--color-accent`      | Pass indicator, links    | ≥ 4.5:1 on `--color-bg` |
| `--color-warn`        | Fail / blocked indicator | ≥ 4.5:1 on `--color-bg` |
| `--color-ghost-fill`  | Ghost SVG fill           | decorative; no text on it|

### `SiteSeancePage`

Top-level page for Site Séance. Owns the URL input field, the submit handler, and the
orchestration of the proxy fetch + core check pipeline. Passes results to `CheckList`,
`GhostCanvas`, and `HauntingScore`.

```ts
// Internal state (not exported)
type SiteSeancePhase =
  | "idle"
  | "fetching"
  | "scoring"
  | "done"
  | "error";
```

### `SkipLink`

A visually hidden `<a href="#main-content">Skip to main content</a>` that becomes visible on
focus. Required for keyboard and screen-reader users.

### `ErrorBoundary`

Catches React render errors in any subtree and renders a plain fallback message. Prevents a
ghost-animation crash from breaking the results view.

---

## Data Models

All models are in-memory only. Nothing is persisted.

```ts
// Produced by packages/core after running all checks.
// Imported type; defined in packages/core/src/scoring/types.ts.
export interface ScoredResult {
  score: number;           // 0–100, integer
  checks: CheckDetail[];
}

export interface CheckDetail {
  id: string;              // e.g. "robots.GPTBot"
  name: string;            // human label
  passed: boolean;
  weight: number;          // contribution to score
  evidence: EvidenceRef;
  category: "robots" | "llms-txt" | "schema" | "meta" | "text";
}

export interface EvidenceRef {
  /** Inline raw text shown in an expandable <details>. */
  raw: string;
  /** Anchor id for the inline evidence block, used as evidenceHref. */
  anchorId: string;
}
```

```ts
// Top-level UI state (held in App.tsx useState).
type AppPhase = "home" | "site-seance" | "ai-seance";

interface AppState {
  phase: AppPhase;
  siteSeanceResult: ScoredResult | null;
  siteSeanceError: string | null;
  // AI Séance state is managed within AiSeancePage (see ai-seance design).
}
```

Ghost level derivation (pure function, testable):

```ts
export function scoreToGhostLevel(score: number): 0 | 1 | 2 | 3 | 4 {
  if (score < 20) return 0;
  if (score < 40) return 1;
  if (score < 60) return 2;
  if (score < 80) return 3;
  return 4;
}
```

Flavour text derivation (pure function, testable):

```ts
export function scoreToFlavourText(score: number): string;
// Returns one of five fixed strings keyed to the same score bands as ghostLevel.
```

---

## Correctness Properties

Properties suitable for fast-check property-based tests. The pure functions targeted are those
defined in `apps/web/src/haunted-ui/` or `packages/core/src/scoring/`. Each property is an
invariant that must hold for all valid inputs.

### P1 — Ghost level is bounded
`scoreToGhostLevel(score)` returns a value in {0, 1, 2, 3, 4} for every integer `score` in
[0, 100].
_Requirements: 1.1_

### P2 — Ghost level is monotone non-decreasing
For any two integer scores `a ≤ b` both in [0, 100],
`scoreToGhostLevel(a) ≤ scoreToGhostLevel(b)`.
_Requirements: 1.1_

### P3 — Flavour text is always a non-empty string
`scoreToFlavourText(score)` returns a non-empty string for every integer `score` in [0, 100].
_Requirements: 1.1_

### P4 — Score bounds are preserved end-to-end
`ScoredResult.score` is always an integer in [0, 100] for any combination of check pass/fail
outcomes. (Wraps the core scoring bounded-score property from Site Séance Requirement 7.2.)
_Requirements: 1.1_

### P5 — Score monotonicity preserved in UI
Changing any single check in a `ScoredResult` from `passed: false` to `passed: true` and
re-scoring does not decrease `score`. (Delegates to core scoring; tested here to confirm the
UI does not transform the score.)
_Requirements: 1.1_ (cross-ref Site Séance Requirement 7.3)

### P6 — Reduced-motion disables animation classes
When the `prefers-reduced-motion` media query is active, `GhostCanvas` renders with no CSS
animation class names on any element. Verified by inspecting the rendered DOM in a jsdom
environment with the media query mocked.
_Requirements: 2.2_

### P7 — Ghost level change is announced
When `ghostLevel` changes to a new value `n`, the `aria-label` prop passed to `GhostCanvas`
contains the string `n` and a denominator (e.g. "Ghost level n of 4"), so screen readers
announce the change via the parent's `aria-live` region.
_Requirements: 1.2, 2.3_

### P8 — CheckEvidence evidence link is keyboard-reachable
Every `CheckEvidence` rendered in a `CheckList` has exactly one `<a>` element whose `href`
is non-empty, ensuring keyboard Tab order reaches the evidence.
_Requirements: 2.3_

### P9 — No third-party names in component output
The rendered output of every Haunted UI component contains none of the strings in a hardcoded
deny-list of AI provider trademarks (e.g. "OpenAI", "Anthropic", "Google"). Provider names
may only appear as raw model-id strings passed down from AI Séance.
_Requirements: 3.1_

### P10 — Judge path reaches results in ≤ 2 interactions
Given a mounted `JudgePath` component, calling `onRunSample()` once reaches `SiteSeancePage`
in scoring or done phase (simulated), satisfying the two-click requirement. Verified by a
React Testing Library interaction test parameterised over the sample URL.
_Requirements: 4.1_

---

## Error Handling

| Scenario | Component | Behaviour |
|---|---|---|
| Proxy fetch fails (network/timeout) | `SiteSeancePage` | Shows inline error banner with plain-language message; Run button re-enabled. |
| URL scheme is not http/https | `SiteSeancePage` | Input shows inline validation error before fetch; no proxy call made. |
| Proxy rejects private-IP URL | `SiteSeancePage` | Shows "This address is not reachable from the checker" with no stack trace. |
| Individual resource returns 404 | `CheckEvidence` | Row shows "not found" as evidence; check is marked failed; scoring continues. |
| Core library throws unexpected error | `ErrorBoundary` | Renders fallback UI; remaining page structure stays intact. |
| Ghost SVG fails to render | `GhostCanvas` | Renders a plain text fallback ("👻") inside an `<svg aria-label>` fallback element. |
| `scoreToGhostLevel` called with out-of-range value | pure function | Clamps to 0 (below range) or 4 (above range) with a `console.warn` in development. |
| Reduced-motion media query change after mount | `FogLayer`, `GhostCanvas` | Both listen to `matchMedia` `change` event and update state; no page reload required. |

Error messages shown in the UI never include stack traces, raw HTTP headers, or any value that
resembles a secret token.

---

## Testing Strategy

### Unit tests (`apps/web/src/haunted-ui/*.test.ts`)

- `scoreToGhostLevel.test.ts`: boundary values (0, 19, 20, 39, 40, 59, 60, 79, 80, 100);
  confirm return type is always in {0,1,2,3,4}.
- `scoreToFlavourText.test.ts`: one assertion per score band; non-empty string output.

### Property-based tests (`apps/web/src/haunted-ui/*.pbt.test.ts`)

- `ghostLevel.pbt.test.ts`: covers P1, P2, P3 using `fc.integer({min:0, max:100})`.
- `scoring.pbt.test.ts`: covers P4, P5 using `fc.record` of arbitrary check pass/fail arrays;
  delegates to core but asserts no UI transformation changes the value.

### Component tests (`apps/web/src/haunted-ui/*.test.tsx`, via Vitest + React Testing Library)

- `GhostCanvas.test.tsx`: all five ghost levels render the correct SVG layers; aria-label
  contains the level number; no animation classes present when reduced-motion is mocked (P6, P7).
- `FogLayer.test.tsx`: does not render when `disabled` is true or reduced-motion is mocked (P6).
- `HauntingScore.test.tsx`: aria-live region present; displays clamped score value.
- `CheckEvidence.test.tsx`: evidence link is rendered and has non-empty href (P8); pass/fail
  conveyed by both colour class and icon text (not colour alone).
- `CheckList.test.tsx`: all items reachable by Tab order in rendered DOM; `onCheckPass` callback
  fires when a check transitions to passed.
- `JudgePath.test.tsx`: onRunSample called on first button click; onOpenDemo called on second
  button click; total interactions ≤ 2 from mount (P10).
- `ThemeProvider.test.tsx`: CSS custom properties are present on the wrapper element; no
  third-party trademark strings appear in any rendered child text (P9).

### Accessibility audit (automated, runs in CI)

- `axe-core` (via `vitest-axe` or `@axe-core/react`) runs against the full mounted
  `SiteSeancePage` with a mock `ScoredResult`; zero violations expected.
- Contrast ratios are verified programmatically for `--color-text-primary`, `--color-text-muted`,
  `--color-accent`, `--color-warn` against `--color-bg` and `--color-surface`.

### Integration smoke test

- Mount `App` with a mocked proxy that returns a pre-recorded HTTP fixture for `canlah.ai`.
- Assert that `ScoredResult.score` is in [0,100], `GhostCanvas` renders, `CheckList` is
  non-empty, and no third-party trademark appears in the output (P9).

### No-secrets CI check

Shared with the repo-level pre-commit hook: grep for `sk-`, `AIza`, `ghp_`, `gho_`, `xox`,
`AKIA`, and PEM headers in all committed files under `apps/web/`.
