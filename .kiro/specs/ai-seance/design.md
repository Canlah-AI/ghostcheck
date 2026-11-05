# Design: AI Séance

## Overview

AI Séance is an optional, client-side feature that asks AI model APIs buying-intent questions and
reports whether a brand appears in the answers. It requires no server involvement beyond what the
AI provider itself exposes; the GhostCheck Worker is never involved in AI Séance requests.

Two execution paths exist:

- **Live mode** — the user supplies their own API key. Requests go directly from the browser to
  the provider's HTTPS endpoint. The key never leaves the browser tab.
- **Demo mode** — no key is required. A static JSON fixture from `/demo` is replayed verbatim.
  Demo data uses canlah.ai or clearly fictional brands only.

Because all network calls in live mode are browser-to-provider, AI Séance has no backend component
of its own and is fully stateless from GhostCheck's perspective.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Browser tab (apps/web)                             │
│                                                     │
│  ┌──────────────┐    ┌────────────────────────────┐ │
│  │ QuestionEditor│    │  SeanceRunner              │ │
│  │  (React)     │───▶│  (React + hooks)           │ │
│  └──────────────┘    └──────────┬─────────────────┘ │
│                                 │                   │
│              ┌──────────────────┼──────────┐        │
│              ▼                  ▼          ▼        │
│      ┌──────────────┐  ┌──────────────┐            │
│      │ demoLoader   │  │ providerClient│            │
│      │ (pure TS)    │  │ (pure TS)    │            │
│      │ /demo/*.json │  │ provider API │            │
│      └──────┬───────┘  └──────┬───────┘            │
│             │                 │                    │
│             └────────┬────────┘                    │
│                      ▼                             │
│             ┌─────────────────┐                    │
│             │  mentionDetector │                    │
│             │  (packages/core) │                    │
│             └─────────────────┘                    │
│                      │                             │
│                      ▼                             │
│             ┌─────────────────┐                    │
│             │  ResultsPanel   │                    │
│             │  (React)        │                    │
│             └─────────────────┘                    │
└─────────────────────────────────────────────────────┘

No GhostCheck Worker involvement in AI Séance.
API key stays in JS heap only — never leaves the tab.
```

Key constraints from steering:

- `packages/core` has no network or DOM access. `mentionDetector` and question template expansion
  live there as pure functions.
- `apps/web` owns all React components, hooks, and the provider client.
- `/demo` owns static fixture files.

---

## Components and Interfaces

### 1. `packages/core/src/ai-seance/questionTemplates.ts`

Pure function that fills template slots with user-supplied values.

```ts
export interface QuestionInput {
  brand: string;      // e.g. "Acme Café"
  category: string;  // e.g. "specialty coffee"
  city: string;       // e.g. "Singapore"
}

// Returns 5–10 filled question strings.
export function generateQuestions(input: QuestionInput): string[];
```

Templates are plain strings stored in the same file (or a sibling `.ts` constants file). No
network access. Output length is clamped to [5, 10].

### 2. `packages/core/src/ai-seance/mentionDetector.ts`

Pure function; no network or DOM.

```ts
export interface MentionResult {
  mentioned: boolean;
  // 1-based position among named businesses in the answer; null if not mentioned.
  position: number | null;
  // All business names detected in the answer, in order of appearance.
  otherNames: string[];
}

export interface DetectionInput {
  answer: string;
  brand: string;
  // Bare domain, e.g. "acme.com" — matched case-insensitively alongside brand name.
  domain?: string;
}

export function detectMention(input: DetectionInput): MentionResult;
```

Detection rules:
- Case-insensitive substring search for `brand` and (if provided) `domain`.
- Named-business extraction uses a simple heuristic: capitalised multi-word noun phrases
  separated by commas, numbered lists, or "and/or" conjunctions. No external NLP dependency.
- `position` is the 1-based index of the brand in `otherNames` (inclusive); `null` when absent.

### 3. `apps/web/src/ai-seance/providerClient.ts`

Thin adapter that maps a question string + provider config to a plain answer string. The API key
is passed in at call time from in-memory state and is never persisted.

```ts
export type SupportedProvider = "openai" | "anthropic";

export interface ProviderConfig {
  provider: SupportedProvider;
  modelId: string;    // exact model id, e.g. "gpt-4o-mini"
  apiKey: string;     // in-memory only; caller's responsibility
}

export interface AskResult {
  modelId: string;
  answer: string;
  // ISO-8601 timestamp of the response, for display only.
  respondedAt: string;
}

// Returns resolved answer or throws ProviderError.
export async function askQuestion(
  question: string,
  config: ProviderConfig,
): Promise<AskResult>;

export class ProviderError extends Error {
  constructor(
    public readonly code: "network" | "auth" | "rate_limit" | "model_not_found" | "unknown",
    message: string,
  ) { super(message); }
}
```

- Calls the provider's chat-completion endpoint directly from the browser.
- Sets a 30-second client-side timeout.
- Never logs or serialises `apiKey`.

### 4. `apps/web/src/ai-seance/demoLoader.ts`

Loads a static fixture; no network call to GhostCheck Worker.

```ts
export interface DemoSample {
  recordedAt: string;          // ISO-8601 date string
  label: "Demo sample";
  brand: string;
  modelId: string;
  questions: DemoQuestionResult[];
}

export interface DemoQuestionResult {
  question: string;
  answer: string;
  mention: MentionResult;      // pre-computed, matches mentionDetector output
}

// Loads /demo/ai-seance-sample.json via fetch (static asset).
export async function loadDemoSample(): Promise<DemoSample>;
```

### 5. `apps/web/src/ai-seance/` React components

| Component | Responsibility |
|---|---|
| `QuestionEditor` | Renders editable list of questions. Add / delete / edit. Controlled by parent state. |
| `KeyEntry` | Password-type input for API key. Value stored in `useState` only. Clears on unmount. |
| `SeanceRunner` | Orchestrates live mode: iterates questions, calls `askQuestion`, streams progress. |
| `ResultsPanel` | Renders `MentionResult` per question. Shows model id. Shows honesty disclaimer. |
| `DemoPanel` | Renders `DemoSample` fixture. Shows "Demo sample" label and `recordedAt` date. |
| `AiSeancePage` | Top-level page component. Routes between live and demo modes. |

---

## Data Models

### `SeanceState` (in-memory React state only)

```ts
type SeancePhase =
  | "idle"
  | "editing-questions"
  | "awaiting-key"
  | "running"
  | "done"
  | "error"
  | "demo";

interface SeanceState {
  phase: SeancePhase;
  brand: string;
  category: string;
  city: string;
  questions: string[];              // 5–10 items after generation
  providerConfig: Omit<ProviderConfig, "apiKey"> | null;
  // apiKey is never stored here; held in KeyEntry component state only
  results: QuestionRunResult[];     // grows as answers arrive
  demoSample: DemoSample | null;
  error: string | null;
}

interface QuestionRunResult {
  question: string;
  modelId: string;
  answer: string;
  mention: MentionResult;
  respondedAt: string;
}
```

### Demo fixture schema (`/demo/ai-seance-sample.json`)

Matches `DemoSample` above. No real brand names, no real API keys, no real answers verbatim from
an unreproducible session. The fixture is a human-authored plausible sample.

---

## Correctness Properties

These are suitable for fast-check property-based tests in `packages/core`. Each property is
stated as an invariant followed by the requirement it covers.

### P1 — Question count bounds
`generateQuestions(input)` always returns an array of length in [5, 10] for any non-empty brand,
category, and city strings.
_Requirements: 1.1_

### P2 — Questions contain brand name
Every generated question string contains the `brand` value (case-insensitive substring match).
_Requirements: 1.1_

### P3 — Mention detection is case-insensitive
For any `brand` string `b` and any answer that contains `b.toUpperCase()` (or any mixed-case
variant), `detectMention` returns `mentioned: true`.
_Requirements: 3.1_

### P4 — Domain match implies mention
If the answer contains the bare `domain` string (case-insensitive), `detectMention` returns
`mentioned: true`.
_Requirements: 3.1_

### P5 — Position is within otherNames bounds
When `mentioned` is `true`, `position` is a positive integer ≤ `otherNames.length`.
_Requirements: 3.1_

### P6 — Not-mentioned ↔ null position
`mentioned === false` if and only if `position === null`.
_Requirements: 3.1_

### P7 — otherNames contains brand when mentioned
When `mentioned === true`, `otherNames` includes a string that case-insensitively matches `brand`
(or `domain` when only the domain triggered the match).
_Requirements: 3.1_

### P8 — Empty answer → not mentioned
`detectMention({ answer: "", brand: anyNonEmpty })` returns `{ mentioned: false, position: null, otherNames: [] }`.
_Requirements: 3.1_

### P9 — Demo sample shape validity
Any JSON that satisfies `DemoSample` has `label === "Demo sample"`, a non-empty `recordedAt` ISO
date, and `questions.length` in [1, 20]. This is a schema-level property that guards fixture
correctness.
_Requirements: 4.1, 4.2_

### P10 — Generated questions are non-empty strings
Every element of `generateQuestions(input)` is a non-empty string with no leading/trailing
whitespace.
_Requirements: 1.1_

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Provider returns HTTP 401 | `ProviderError("auth", ...)`. UI shows "Invalid or expired API key" and prompts re-entry. Does not clear other results. |
| Provider returns HTTP 429 | `ProviderError("rate_limit", ...)`. UI pauses the run and shows a backoff message with remaining questions count. |
| Network failure / timeout | `ProviderError("network", ...)`. UI marks the individual question as failed and continues with remaining questions. |
| Unknown HTTP error | `ProviderError("unknown", ...)`. UI shows the HTTP status code for the user to investigate. |
| Demo fixture fetch fails | `demoLoader` throws. UI shows "Demo unavailable" and hides the demo button. Does not affect live mode. |
| Brand/category/city empty | `generateQuestions` throws `RangeError`. UI blocks the "Generate" button if any field is empty (validation before call). |
| Question list < 1 after edits | UI blocks the "Run" button if the question list is empty. |
| API key field empty | UI blocks the "Run" button; the key input shows an inline validation message. |

No errors are logged to any external service. Error messages shown to the user never echo the API
key back.

---

## Testing Strategy

### Unit tests (`*.test.ts` in `packages/core/src/ai-seance/`)

- `questionTemplates.test.ts`: verify each template slot is filled; verify count is in [5, 10]
  for a set of representative inputs; verify output contains the brand name.
- `mentionDetector.test.ts`: exact match, case-insensitive match, domain match, no match,
  multi-business answer with position verification, empty answer.

### Property-based tests (`*.pbt.test.ts` in `packages/core/src/ai-seance/`)

- `questionTemplates.pbt.test.ts`: covers P1, P2, P10 using fast-check arbitrary strings for
  brand / category / city (filtered to non-empty).
- `mentionDetector.pbt.test.ts`: covers P3, P4, P5, P6, P7, P8 using fast-check arbitrary
  brand strings, answer strings, and injected brand substrings.

### Component tests (`apps/web/src/ai-seance/`)

- `KeyEntry.test.tsx`: confirm that the value is never exposed outside the component via props or
  DOM attributes other than the password input itself.
- `QuestionEditor.test.tsx`: add / delete / edit round-trips; verify count constraints are
  surfaced to the parent.
- `DemoPanel.test.tsx`: renders "Demo sample" label and `recordedAt` date; renders disclaimer.
- `ResultsPanel.test.tsx`: renders the model id; renders the honesty disclaimer string for every
  result; renders "not mentioned" state correctly.

### Integration / smoke tests

- Load the demo fixture via `demoLoader` in a jsdom environment and assert it validates against
  `DemoSample` schema (covers P9).
- `providerClient` is not unit-tested with live API calls. It is covered by a manual smoke test
  checklist in `docs/smoke-tests.md` that verifies key-in-memory behaviour using browser DevTools.

### No-secrets check

Secret scanning is enforced by the `secret-scan` Kiro hook (`.kiro/hooks/secret-scan.kiro.hook`), which runs automatically on `agentStop` and on demand. It greps changed files for `sk-`, `AIza`, `ghp_`, `gho_`, `xox`, `AKIA`, and PEM headers, allowing only the literal placeholder `sk-REDACTED` in test fixtures. There is no separate CI pipeline.
