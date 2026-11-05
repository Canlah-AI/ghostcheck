# Tasks: Haunted UI

Implementation plan. Each task is self-contained and ordered so that dependencies are satisfied
before they are needed. Complete tasks in order unless otherwise noted.

---

- [ ] 1. Set up `apps/web` Vite + React scaffold
  - Initialise the `apps/web` npm workspace with `vite`, `react`, `react-dom`, and their
    TypeScript type packages at pinned versions.
  - Add `tsconfig.json` with `strict: true`, `jsx: react-jsx`, and path aliases for
    `@core` → `../../packages/core/src`.
  - Add a minimal `index.html` entry and a `src/main.tsx` that mounts `<App />`.
  - Confirm `npm run dev` and `npm run build` succeed with the empty scaffold.
  - _Requirements: 4.1_

- [ ] 2. Create `ThemeProvider` with spooky WCAG-AA palette
  - Define CSS custom properties (`--color-bg`, `--color-surface`, `--color-text-primary`,
    `--color-text-muted`, `--color-accent`, `--color-warn`, `--color-ghost-fill`) in a
    global stylesheet imported by `ThemeProvider.tsx`.
  - Choose colour values that achieve ≥ 4.5:1 contrast (normal text) and ≥ 3:1 (large text /
    UI components) against their respective backgrounds. Document the ratios in a comment.
  - Write `ThemeProvider.test.tsx`: assert that each CSS custom property is present on the
    wrapper element; assert no third-party trademark strings appear in rendered output.
  - _Requirements: 2.1, 3.1_

- [ ] 3. Add `SkipLink` and `ErrorBoundary` shared components
  - `SkipLink.tsx`: visually-hidden `<a href="#main-content">Skip to main content</a>` that
    becomes visible on focus; add a matching `id="main-content"` anchor in `App.tsx`.
  - `ErrorBoundary.tsx`: class component that catches render errors and shows a plain fallback
    message without stack traces.
  - Write `SkipLink.test.tsx`: assert the link has `href="#main-content"` and is focusable.
  - Write `ErrorBoundary.test.tsx`: assert fallback renders when a child throws.
  - _Requirements: 2.3_

- [ ] 4. Implement `FogLayer` component
  - `FogLayer.tsx`: CSS-animated fog overlay, absolutely positioned, behind all content.
  - On mount, read `window.matchMedia('(prefers-reduced-motion: reduce)')`.
    If true, or if `disabled` prop is true, do not render the animated element.
  - Subscribe to the `matchMedia` `change` event and update state dynamically.
  - Write `FogLayer.test.tsx`:
    - Renders nothing when `disabled={true}`.
    - Renders nothing when the reduced-motion media query is mocked as active.
    - Renders the fog element when both conditions are false.
  - _Requirements: 2.2_

- [ ] 5. Implement `scoreToGhostLevel` and `scoreToFlavourText` pure functions
  - Create `apps/web/src/haunted-ui/ghostLevel.ts` exporting:
    - `scoreToGhostLevel(score: number): 0 | 1 | 2 | 3 | 4` — maps score bands to level;
      clamps out-of-range inputs and emits `console.warn` in development.
    - `scoreToFlavourText(score: number): string` — returns one of five fixed flavour strings
      keyed to the same bands.
  - Write `scoreToGhostLevel.test.ts`: boundary values 0, 19, 20, 39, 40, 59, 60, 79, 80, 100.
  - Write `scoreToFlavourText.test.ts`: one assertion per band; non-empty string.
  - Write `ghostLevel.pbt.test.ts` covering P1, P2, P3 using `fc.integer({min:0, max:100})`.
  - _Requirements: 1.1_

- [ ] 6. Implement `GhostCanvas` SVG component
  - `GhostCanvas.tsx`: SVG ghost with five layers (body outline, eyes, smile, glow, full form).
    Each layer is visible when `ghostLevel >= layerThreshold`.
  - Accept `ghostLevel: 0|1|2|3|4` and `ariaLabel: string` props.
  - Layer transitions use CSS `opacity` / `scale` with a short duration. Apply no animation
    class when `prefers-reduced-motion` is active (mirror FogLayer's media-query subscription).
  - Include a plain-text fallback ("👻") in an `<aria-label>`-carrying `<title>` element.
  - Write `GhostCanvas.test.tsx`:
    - All five levels render correct layers.
    - `ariaLabel` prop appears on the root `<svg>` element.
    - No animation class names present when reduced-motion media query is mocked (P6, P7).
  - _Requirements: 1.1, 1.2, 2.2, 2.3_

- [ ] 7. Implement `HauntingScore` component
  - `HauntingScore.tsx`: displays an integer score (0–100) inside an SVG progress ring.
  - Non-reduced-motion: count-up animation from 0 to final score.
  - Reduced-motion: value renders immediately at final score.
  - Wrapping `<div>` carries `aria-live="polite"` and an `aria-label` with the score value.
  - Renders `flavourText` below the ring.
  - Write `HauntingScore.test.tsx`:
    - `aria-live` region is present.
    - Clamped score values (0, 100) render without error.
    - `flavourText` appears in the DOM.
  - _Requirements: 1.1, 2.3_

- [ ] 8. Implement `CheckEvidence` component
  - `CheckEvidence.tsx`: one result row with check name, pass/fail indicator (icon + colour,
    never colour alone), short summary, and an `<a>` evidence link.
  - Pass/fail icon uses a Unicode character or inline SVG with `aria-hidden="true"`;
    pass/fail state is also conveyed via a visually-hidden `<span>` for screen readers.
  - Weight is shown in a `<details>/<summary>` tooltip.
  - Write `CheckEvidence.test.tsx`:
    - Evidence link `href` is non-empty and the link is in the Tab order (P8).
    - Both icon and visually-hidden text convey pass/fail state.
    - Weight value appears in the DOM.
  - _Requirements: 1.2, 2.3_

- [ ] 9. Implement `CheckList` component
  - `CheckList.tsx`: renders an ordered list grouped by category with `<h3>` headings.
  - Accepts `checks: CheckResultSummary[]` and optional `onCheckPass` callback.
  - When a check item transitions from `passed: false` to `passed: true` between renders,
    call `onCheckPass(checkName)`.
  - Write `CheckList.test.tsx`:
    - All items Tab-reachable in rendered DOM.
    - `onCheckPass` fires exactly once per transitioned check.
    - Groups render correct `<h3>` headings.
  - _Requirements: 1.2, 2.3_

- [ ] 10. Implement `JudgePath` component
  - `JudgePath.tsx`: two-button entry widget.
    - Button 1 ("Run sample") — pre-fills URL input and calls `onRunSample()`.
    - Button 2 ("Try demo mode") — calls `onOpenDemo()`.
  - Accepts `sampleUrl`, `onRunSample`, `onOpenDemo` props.
  - Both buttons are keyboard-focusable and have descriptive `aria-label` text.
  - Write `JudgePath.test.tsx`:
    - `onRunSample` is called on the first button click (P10).
    - `onOpenDemo` is called on the second button click.
    - Both buttons are focusable via Tab.
  - _Requirements: 4.1_

- [ ] 11. Implement `SiteSeancePage` — URL input and fetch orchestration
  - `SiteSeancePage.tsx`: text input for URL, submit button, phase state machine
    (`idle` → `fetching` → `scoring` → `done` | `error`).
  - Before fetching, validate URL scheme (http/https only); show inline validation error if
    invalid, do not call the proxy.
  - On submit, POST to the Cloudflare Worker proxy; handle network error, timeout, and
    proxy-level rejection (private IP) with distinct inline error messages.
  - On success, pass raw response bodies to `packages/core` check pipeline and receive
    `ScoredResult`.
  - Wire `ScoredResult` into `CheckList`, `GhostCanvas` (via `scoreToGhostLevel`), and
    `HauntingScore` (via `scoreToFlavourText`).
  - Include `JudgePath` above the input, pre-configured with the `canlah.ai` sample URL.
  - Write `SiteSeancePage.test.tsx`:
    - Invalid URL scheme shows validation error; no fetch call made.
    - Proxy rejection (mocked) shows "not reachable" message.
    - Successful mocked response renders `CheckList` and `HauntingScore`.
  - _Requirements: 1.1, 1.2, 2.3, 4.1_

- [ ] 12. Wire ghost animation to check passage events
  - In `SiteSeancePage`, track which checks have transitioned to passed since the last render.
  - When `CheckList.onCheckPass` fires, update a state list of recently-passed check names.
  - Render a transient `<div aria-live="assertive">` that announces the passing check name
    to screen readers.
  - GhostCanvas re-renders with updated `ghostLevel` derived from the current score.
  - Write a test asserting the `aria-live` region is populated when `onCheckPass` fires.
  - _Requirements: 1.2, 2.3_

- [ ] 13. Integrate `AiSeancePage` into `App` routing
  - `App.tsx` manages `AppState.phase` (`"home"` | `"site-seance"` | `"ai-seance"`).
  - Navigation between phases is driven by buttons (no URL router required for V1).
  - `ThemeProvider` and `FogLayer` wrap the entire app.
  - `SkipLink` is rendered as the first child, before `ThemeProvider`.
  - `ErrorBoundary` wraps `SiteSeancePage` and `AiSeancePage` independently.
  - Write `App.test.tsx`:
    - Starting phase is `"home"`.
    - Clicking the Site Séance entry button transitions to `"site-seance"`.
    - Clicking the AI Séance / demo entry transitions to `"ai-seance"`.
  - _Requirements: 4.1_

- [ ] 14. Run automated accessibility audit in CI
  - Install `vitest-axe` (or `@axe-core/react`) at a pinned version.
  - Write `accessibility.test.tsx`:
    - Mount `SiteSeancePage` with a mock `ScoredResult` fixture; assert zero axe violations.
    - Mount `JudgePath` standalone; assert zero axe violations.
  - Verify programmatically that `--color-text-primary` and `--color-text-muted` against
    `--color-bg`, and `--color-accent` and `--color-warn` against `--color-bg`, each satisfy
    4.5:1. Encode the ratios as constants and assert `ratio >= 4.5` in the test.
  - _Requirements: 2.1, 2.3_

- [ ] 15. Add no-secrets CI check for `apps/web`
  - Extend the repo-level pre-commit hook (or add a Vitest global setup file) to grep all
    files under `apps/web/` for patterns `sk-`, `AIza`, `ghp_`, `gho_`, `xox`, `AKIA`, and
    PEM headers.
  - Fail the check if any match is found outside a fixture file containing only the pattern
    as a redacted placeholder (e.g., `"sk-REDACTED"`).
  - Confirm the check passes on the current codebase.
  - _Requirements: 2.3_ (cross-ref security steering rule 1)

- [ ] 16. Integration smoke test: full Site Séance render with `canlah.ai` fixture
  - In `apps/web/src/__tests__/smoke.test.tsx`, mount `App` with a mocked Worker proxy that
    returns pre-recorded HTTP fixtures for `https://canlah.ai`.
  - Assert:
    - `ScoredResult.score` is an integer in [0, 100].
    - `GhostCanvas` is present in the DOM.
    - `CheckList` contains at least one item.
    - No third-party trademark strings appear anywhere in the rendered output (P9).
  - Clean up any temporary fixture files after the test run.
  - _Requirements: 1.1, 3.1, 4.1_

- [ ] 17. Build and deploy verification
  - Run `npm run build` in `apps/web`; confirm zero TypeScript errors and zero Vite build
    errors.
  - Run the full Vitest suite (`npm test`); confirm all tests pass.
  - Verify the built `dist/` directory contains only static assets (no secrets, no private
    data).
  - Confirm `FogLayer` is absent from the built output when the fixture HTML sets
    `prefers-reduced-motion: reduce` (manual check in browser DevTools).
  - _Requirements: 2.1, 2.2, 2.3, 4.1_
