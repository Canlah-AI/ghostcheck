# Tasks: AI Séance

## Implementation Plan

Numbered checkbox implementation plan. Complete tasks in order. Each task ends with the
requirement(s) it satisfies.

---

- [ ] 1. Scaffold the `ai-seance` module in `packages/core`
  - [ ] 1.1 Create `packages/core/src/ai-seance/` with an `index.ts` barrel that exports `generateQuestions`, `detectMention`, and their public types.
  - [ ] 1.2 Verify TypeScript strict-mode compilation passes with no errors (`tsc --noEmit`).

  _Requirements: 1.1, 3.1_

---

- [ ] 2. Implement and test question templates (`packages/core/src/ai-seance/questionTemplates.ts`)
  - [ ] 2.1 Define `QuestionInput` (`brand`, `category`, `city`) and implement `generateQuestions(input: QuestionInput): string[]` with ≥ 10 template strings, output clamped to [5, 10], throwing `RangeError` if any field is empty or whitespace-only.
  - [ ] 2.2 Unit tests: output length ∈ [5, 10], each string contains brand name, `RangeError` on empty fields.
  - [ ] 2.3 Property-based tests: P1 (length bounds), P2 (brand substring), P10 (non-empty trimmed strings).

  _Requirements: 1.1_

---

- [ ] 3. Implement and test mention detector (`packages/core/src/ai-seance/mentionDetector.ts`)
  - [ ] 3.1 Define `DetectionInput` / `MentionResult`; implement case-insensitive `brand` + `domain` search; extract business names via capitalised noun-phrase heuristic; `position` is 1-based index, `null` when absent; empty answer → `{ mentioned: false, position: null, otherNames: [] }`.
  - [ ] 3.2 Unit tests: exact match, case-insensitive, domain match, no match, multi-business position, empty answer.
  - [ ] 3.3 PBT: P3 (uppercase brand always matches), P4 (domain implies mention), P5 (position in bounds), P6 (not-mentioned ↔ null position), P7 (otherNames contains brand when mentioned), P8 (empty answer always not-mentioned).

  _Requirements: 3.1_

---

- [ ] 4. Author demo fixture and implement demo loader
  - [ ] 4.1 Create `/demo/ai-seance-sample.json` using the `DemoSample` schema: `label: "Demo sample"`, ISO-8601 `recordedAt`, `brand` (canlah.ai or fictional), `modelId`, 5–10 `DemoQuestionResult` entries with human-authored answers; `mention` fields pre-computed to match `detectMention` output.
  - [ ] 4.2 Implement `apps/web/src/ai-seance/demoLoader.ts`: `loadDemoSample(): Promise<DemoSample>` via static fetch; validate shape; throw descriptive error on mismatch.
  - [ ] 4.3 Integration test: assert `label === "Demo sample"`, `recordedAt` is parseable ISO date, `questions.length ∈ [1, 20]` (P9), no answer contains secret patterns.

  _Requirements: 4.1, 4.2_

---

- [ ] 5. Implement provider client (`apps/web/src/ai-seance/providerClient.ts`)
  - [ ] 5.1 Define `SupportedProvider` (`"openai" | "anthropic"`), `ProviderConfig`, `AskResult`, `ProviderError` (codes: `network | auth | rate_limit | model_not_found | unknown`). Implement `askQuestion(question, config): Promise<AskResult>` with direct browser-to-provider HTTPS fetch, 30-second `AbortController` timeout, and HTTP error mapping. Never log or serialise `apiKey`.
  - [ ] 5.2 Manual smoke-test checklist in `docs/smoke-tests.md`: confirm via browser DevTools that no API key appears in Network requests to the GhostCheck Worker.

  _Requirements: 2.1, 2.2, 2.3, 2.4_

---

- [ ] 6. Build `KeyEntry` component (`apps/web/src/ai-seance/KeyEntry.tsx`)
  - [ ] 6.1 Render `<input type="password">`; hold value in `useState` only; expose via `onRun` callback at press time; clear on unmount; show inline validation when empty.
  - [ ] 6.2 Unit tests: key not in non-password DOM attributes; "Run" blocked on empty key; `onRun` called with correct key when non-empty.

  _Requirements: 2.1, 2.2_

---

- [ ] 7. Build `QuestionEditor` component (`apps/web/src/ai-seance/QuestionEditor.tsx`)
  - [ ] 7.1 Render questions as editable inputs; per-question "Delete"; "Add question" control; `onChange` on every mutation; visually indicate empty list.
  - [ ] 7.2 Unit tests: edit/delete/add fire `onChange` with updated array; empty-list indicator appears.

  _Requirements: 1.2_

---

- [ ] 8. Build `SeanceRunner` component (`apps/web/src/ai-seance/SeanceRunner.tsx`)
  - [ ] 8.1 Accept `questions`, `providerConfig`, `onResult`; iterate questions sequentially calling `askQuestion` + `detectMention`; show progress ("3 of 7 questions answered"); on `rate_limit` pause with backoff message; on other errors mark question failed and continue.

  _Requirements: 2.3, 2.4, 3.1_

---

- [ ] 9. Build `ResultsPanel` and `DemoPanel` components
  - [ ] 9.1 `ResultsPanel.tsx`: render per-question `QuestionRunResult` (question, answer, `mentioned`, `position`, `otherNames`, `modelId`); "not mentioned" in distinct style; honesty disclaimer once: "An API answer is not the same as what a person sees in a consumer chat app."
  - [ ] 9.2 `DemoPanel.tsx`: render `DemoSample` via `ResultsPanel` sub-components; show "Demo sample" label and human-readable `recordedAt`.
  - [ ] 9.3 Unit tests for each: model id per row; disclaimer present; "not mentioned" renders; "Demo sample" label and date rendered.

  _Requirements: 2.4, 3.1, 3.2, 4.1, 4.2_

---

- [ ] 10. Build top-level `AiSeancePage` (`apps/web/src/ai-seance/AiSeancePage.tsx`)
  - [ ] 10.1 Brand/category/city fields + "Generate questions"; `QuestionEditor`; `KeyEntry` for provider (OpenAI / Anthropic) + key; "Run demo" visible when no key, hidden once key entered; "Run" blocked when question list empty.
  - [ ] 10.2 Live run: render `SeanceRunner` → `ResultsPanel`. Demo mode: render `DemoPanel` (via `loadDemoSample`).

  _Requirements: 1.1, 1.2, 2.1, 2.2, 4.1_

---

- [ ] 11. Wire `AiSeancePage` into the main app and final verification
  - [ ] 11.1 Add navigation route/state and a link from the main page; confirm page loads in live and demo modes via local `vite dev`.
  - [ ] 11.2 `tsc --noEmit` across workspace: zero errors. `vitest run` across `packages/core` and `apps/web`: all tests pass.
  - [ ] 11.3 Confirm with DevTools that no API key appears in Network requests to the GhostCheck Worker; verify demo mode works end-to-end without a key; confirm honesty disclaimer visible on every result view.
  - [ ] 11.4 Confirm no-secrets compliance is enforced by the `secret-scan` Kiro hook (`.kiro/hooks/secret-scan.kiro.hook`) — no separate CI step required.

  _Requirements: 1.1, 2.1, 2.2, 3.2, 4.1, 4.2_
