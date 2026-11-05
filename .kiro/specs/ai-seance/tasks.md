# Tasks: AI Séance

Numbered checkbox implementation plan. Complete tasks in order. Each task ends with the
requirement(s) it satisfies. Sub-tasks are indented under their parent.

---

- [ ] 1. Scaffold the `ai-seance` module in `packages/core`
  - [ ] 1.1 Create `packages/core/src/ai-seance/` directory with an `index.ts` barrel export.
  - [ ] 1.2 Export `generateQuestions`, `detectMention` and their public types from the barrel.
  - [ ] 1.3 Verify TypeScript strict-mode compilation passes with no errors (`tsc --noEmit`).

  _Requirements: 1.1, 3.1_

---

- [ ] 2. Implement question templates (`packages/core/src/ai-seance/questionTemplates.ts`)
  - [ ] 2.1 Define a `QuestionInput` interface with `brand`, `category`, and `city` string fields.
  - [ ] 2.2 Write at least 10 template strings that reference `{brand}`, `{category}`, and `{city}` placeholders.
  - [ ] 2.3 Implement `generateQuestions(input: QuestionInput): string[]` that fills placeholders and returns 5–10 questions (slice to 10 if the template list grows beyond that).
  - [ ] 2.4 Throw `RangeError` if any of `brand`, `category`, or `city` is empty or whitespace-only.

  _Requirements: 1.1_

---

- [ ] 3. Unit-test question templates (`packages/core/src/ai-seance/questionTemplates.test.ts`)
  - [ ] 3.1 Test that a representative set of inputs produces arrays of length in [5, 10].
  - [ ] 3.2 Test that each returned string is non-empty, trimmed, and contains the brand name.
  - [ ] 3.3 Test that empty `brand`, `category`, or `city` throws `RangeError`.

  _Requirements: 1.1_

---

- [ ] 4. Property-based tests for question templates (`packages/core/src/ai-seance/questionTemplates.pbt.test.ts`)
  - [ ] 4.1 Use fast-check to generate arbitrary non-empty brand, category, and city strings.
  - [ ] 4.2 Assert output length is always in [5, 10] (P1).
  - [ ] 4.3 Assert every question contains the brand substring case-insensitively (P2).
  - [ ] 4.4 Assert every question is a non-empty trimmed string (P10).

  _Requirements: 1.1_

---

- [ ] 5. Implement mention detector (`packages/core/src/ai-seance/mentionDetector.ts`)
  - [ ] 5.1 Define `DetectionInput` (`answer`, `brand`, optional `domain`) and `MentionResult` (`mentioned`, `position`, `otherNames`) interfaces.
  - [ ] 5.2 Implement case-insensitive substring search for `brand` and `domain` in `answer`.
  - [ ] 5.3 Implement a heuristic business-name extractor: capitalised multi-word noun phrases separated by commas, numbered lists, or "and"/"or" conjunctions.
  - [ ] 5.4 Set `position` to the 1-based index of the brand in `otherNames`; `null` when absent.
  - [ ] 5.5 Return `{ mentioned: false, position: null, otherNames: [] }` for an empty `answer`.

  _Requirements: 3.1_

---

- [ ] 6. Unit-test mention detector (`packages/core/src/ai-seance/mentionDetector.test.ts`)
  - [ ] 6.1 Test exact brand match → `mentioned: true`, correct position.
  - [ ] 6.2 Test case-insensitive brand match → `mentioned: true`.
  - [ ] 6.3 Test bare domain match → `mentioned: true`.
  - [ ] 6.4 Test answer with no brand → `mentioned: false`, `position: null`.
  - [ ] 6.5 Test multi-business answer to verify `otherNames` order and `position` value.
  - [ ] 6.6 Test empty answer → `{ mentioned: false, position: null, otherNames: [] }`.

  _Requirements: 3.1_

---

- [ ] 7. Property-based tests for mention detector (`packages/core/src/ai-seance/mentionDetector.pbt.test.ts`)
  - [ ] 7.1 P3: injecting `brand.toUpperCase()` into any answer always yields `mentioned: true`.
  - [ ] 7.2 P4: injecting the `domain` string into any answer always yields `mentioned: true`.
  - [ ] 7.3 P5: when `mentioned` is `true`, `position` is a positive integer ≤ `otherNames.length`.
  - [ ] 7.4 P6: `mentioned === false` iff `position === null`.
  - [ ] 7.5 P7: when `mentioned === true`, `otherNames` contains a case-insensitive match for `brand` (or `domain`).
  - [ ] 7.6 P8: empty `answer` always returns `{ mentioned: false, position: null, otherNames: [] }`.

  _Requirements: 3.1_

---

- [ ] 8. Author demo fixture (`/demo/ai-seance-sample.json`)
  - [ ] 8.1 Create the file using the `DemoSample` schema from the design doc.
  - [ ] 8.2 Use only canlah.ai or clearly fictional brand names — no real businesses.
  - [ ] 8.3 Set `label` to `"Demo sample"` and `recordedAt` to the current ISO-8601 date.
  - [ ] 8.4 Include 5–10 `DemoQuestionResult` entries with plausible human-authored answers.
  - [ ] 8.5 Pre-compute each `mention` field so it matches what `detectMention` would return.

  _Requirements: 4.1, 4.2_

---

- [ ] 9. Implement demo loader (`apps/web/src/ai-seance/demoLoader.ts`)
  - [ ] 9.1 Define the `DemoSample` and `DemoQuestionResult` TypeScript interfaces.
  - [ ] 9.2 Implement `loadDemoSample(): Promise<DemoSample>` — static `fetch` of the JSON asset, no GhostCheck Worker involvement.
  - [ ] 9.3 Validate the parsed JSON against the `DemoSample` shape; throw a descriptive error if validation fails.

  _Requirements: 4.1, 4.2_

---

- [ ] 10. Integration test for demo fixture (`apps/web/src/ai-seance/demoLoader.test.ts`)
  - [ ] 10.1 In a jsdom/Vitest environment, call `loadDemoSample()` with the real fixture file.
  - [ ] 10.2 Assert `label === "Demo sample"`, `recordedAt` is a parseable ISO date, and `questions.length` is in [1, 20] (P9).
  - [ ] 10.3 Assert no answer or brand string in the fixture contains known secret patterns (`sk-`, `AIza`, etc.).

  _Requirements: 4.1, 4.2_

---

- [ ] 11. Implement provider client (`apps/web/src/ai-seance/providerClient.ts`)
  - [ ] 11.1 Define `SupportedProvider`, `ProviderConfig` (with `apiKey: string`), `AskResult`, and `ProviderError` types.
  - [ ] 11.2 Implement `askQuestion(question, config): Promise<AskResult>` with direct browser-to-provider HTTPS fetch.
  - [ ] 11.3 Apply a 30-second client-side timeout using `AbortController`.
  - [ ] 11.4 Map HTTP 401 → `ProviderError("auth")`, 429 → `ProviderError("rate_limit")`, network/timeout → `ProviderError("network")`, other → `ProviderError("unknown")`.
  - [ ] 11.5 Include the exact `modelId` from the API response (or config) in `AskResult`.
  - [ ] 11.6 Ensure `apiKey` is never written to any log statement, error message, or serialised object.

  _Requirements: 2.1, 2.2, 2.3, 2.4_

---

- [ ] 12. Build `KeyEntry` component (`apps/web/src/ai-seance/KeyEntry.tsx`)
  - [ ] 12.1 Render a `<input type="password">` for the API key.
  - [ ] 12.2 Hold the value in `useState` only; expose it to the parent only via an `onRun` callback that receives the key at the moment "Run" is pressed.
  - [ ] 12.3 Clear the state on component unmount.
  - [ ] 12.4 Do not echo or expose the key in any attribute, aria label, or DOM node other than the `value` of the password input.
  - [ ] 12.5 Show an inline validation message when the user attempts to run with an empty key.

  _Requirements: 2.1, 2.2_

---

- [ ] 13. Unit-test `KeyEntry` (`apps/web/src/ai-seance/KeyEntry.test.tsx`)
  - [ ] 13.1 Confirm the key value is not present in any non-password DOM attribute.
  - [ ] 13.2 Confirm "Run" is blocked when the key is empty and shows the validation message.
  - [ ] 13.3 Confirm `onRun` is called with the entered key when the key is non-empty.

  _Requirements: 2.1, 2.2_

---

- [ ] 14. Build `QuestionEditor` component (`apps/web/src/ai-seance/QuestionEditor.tsx`)
  - [ ] 14.1 Render the list of questions as editable text inputs.
  - [ ] 14.2 Provide "Delete" per question and an "Add question" control.
  - [ ] 14.3 Surface the current question array to the parent via an `onChange` callback on every mutation.
  - [ ] 14.4 Visually indicate when the list is empty so the parent can block "Run".

  _Requirements: 1.2_

---

- [ ] 15. Unit-test `QuestionEditor` (`apps/web/src/ai-seance/QuestionEditor.test.tsx`)
  - [ ] 15.1 Test that editing a question text fires `onChange` with the updated array.
  - [ ] 15.2 Test that deleting a question removes it and fires `onChange`.
  - [ ] 15.3 Test that adding a question appends a blank entry and fires `onChange`.
  - [ ] 15.4 Test that the empty-list indicator appears when all questions are deleted.

  _Requirements: 1.2_

---

- [ ] 16. Build `SeanceRunner` component (`apps/web/src/ai-seance/SeanceRunner.tsx`)
  - [ ] 16.1 Accept questions array, provider config, and an `onResult` callback as props.
  - [ ] 16.2 Iterate through questions sequentially, calling `askQuestion` for each.
  - [ ] 16.3 Pass each `AskResult` through `detectMention` and call `onResult` with a `QuestionRunResult`.
  - [ ] 16.4 Display per-question progress (e.g., "3 of 7 questions answered").
  - [ ] 16.5 On `ProviderError("rate_limit")`, pause and surface a backoff message rather than aborting the whole run.
  - [ ] 16.6 On other `ProviderError` variants, mark the individual question as failed and continue.

  _Requirements: 2.3, 2.4, 3.1_

---

- [ ] 17. Build `ResultsPanel` component (`apps/web/src/ai-seance/ResultsPanel.tsx`)
  - [ ] 17.1 Render each `QuestionRunResult` with: question text, answer text, `mentioned` status, `position`, `otherNames`, and the exact `modelId` used.
  - [ ] 17.2 Render the honesty disclaimer: "An API answer is not the same as what a person sees in a consumer chat app." on the panel (not per row).
  - [ ] 17.3 Render "not mentioned" in a visually distinct style when `mentioned === false`.

  _Requirements: 2.4, 3.1, 3.2_

---

- [ ] 18. Unit-test `ResultsPanel` (`apps/web/src/ai-seance/ResultsPanel.test.tsx`)
  - [ ] 18.1 Assert the model id string is rendered for every result row.
  - [ ] 18.2 Assert the honesty disclaimer text is present in the rendered output.
  - [ ] 18.3 Assert "not mentioned" state renders without crashing and displays the correct label.

  _Requirements: 2.4, 3.1, 3.2_

---

- [ ] 19. Build `DemoPanel` component (`apps/web/src/ai-seance/DemoPanel.tsx`)
  - [ ] 19.1 Accept a `DemoSample` as a prop and render each `DemoQuestionResult`.
  - [ ] 19.2 Display the "Demo sample" label prominently.
  - [ ] 19.3 Display the `recordedAt` date in a human-readable format.
  - [ ] 19.4 Re-use `ResultsPanel` (or its sub-components) for per-question rendering, including model id and honesty disclaimer.

  _Requirements: 4.1, 4.2_

---

- [ ] 20. Unit-test `DemoPanel` (`apps/web/src/ai-seance/DemoPanel.test.tsx`)
  - [ ] 20.1 Assert "Demo sample" label is rendered.
  - [ ] 20.2 Assert `recordedAt` date is rendered.
  - [ ] 20.3 Assert the honesty disclaimer text is present.

  _Requirements: 4.1, 4.2_

---

- [ ] 21. Build top-level `AiSeancePage` component (`apps/web/src/ai-seance/AiSeancePage.tsx`)
  - [ ] 21.1 Render fields for brand, category, and city with a "Generate questions" button.
  - [ ] 21.2 Render `QuestionEditor` with the generated questions once available.
  - [ ] 21.3 Render `KeyEntry` for provider selection (OpenAI / Anthropic) and key entry.
  - [ ] 21.4 Show a "Run demo" button when no key has been entered; hide it once a key is present.
  - [ ] 21.5 Render `SeanceRunner` when live run is in progress.
  - [ ] 21.6 Render `ResultsPanel` when live results are available.
  - [ ] 21.7 Render `DemoPanel` (loaded via `loadDemoSample`) in demo mode.
  - [ ] 21.8 Block "Run" button if question list is empty.

  _Requirements: 1.1, 1.2, 2.1, 2.2, 4.1_

---

- [ ] 22. Wire `AiSeancePage` into the main app router (`apps/web/src/`)
  - [ ] 22.1 Add a route for `/seance` (or the project's chosen path) that renders `AiSeancePage`.
  - [ ] 22.2 Add a navigation link from the main page to the AI Séance page.
  - [ ] 22.3 Confirm the page loads in both live and demo modes in a local `vite dev` session.

  _Requirements: 1.1, 2.1, 4.1_

---

- [ ] 23. Add no-secrets CI check
  - [ ] 23.1 Add a pre-commit hook (e.g., via a shell script in `.kiro/hooks/` or a lint-staged config) that greps the staged diff for `sk-`, `AIza`, `ghp_`, `gho_`, `xox`, `AKIA`, and PEM headers.
  - [ ] 23.2 Add the same check as a CI step that runs on every pull request and fails if any match is found outside allowed placeholder strings (e.g., `sk-REDACTED`).

  _Requirements: 2.2_

---

- [ ] 24. Final verification pass
  - [ ] 24.1 Run `tsc --noEmit` across the entire workspace; zero errors.
  - [ ] 24.2 Run `vitest run` across `packages/core` and `apps/web`; all tests pass.
  - [ ] 24.3 Manually verify with browser DevTools that no API key appears in Network request headers to the GhostCheck Worker.
  - [ ] 24.4 Manually verify demo mode works end-to-end without any API key.
  - [ ] 24.5 Confirm the honesty disclaimer is visible on every result view.

  _Requirements: 2.2, 3.2, 4.1, 4.2_
