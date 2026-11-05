# Requirements: AI Séance

## Introduction
AI Séance asks AI model APIs buying-intent questions and reports whether a brand is mentioned. It runs only with the user's own API key, or in a demo mode that replays a committed sample.

## Requirements

### Requirement 1: Questions
**User story:** As a marketer, I want realistic buying questions for my category and city, so that the test reflects how customers ask.
1. WHEN the user enters brand, category and city THEN the system SHALL generate 5–10 questions from public templates in the repo.
2. The user SHALL be able to edit, delete or add questions before running.

### Requirement 2: Bring your own key
1. The system SHALL accept a key for 1–2 supported model APIs, entered by the user.
2. The key SHALL be held only in memory for the current tab. It SHALL NOT be sent to the GhostCheck Worker, written to storage or logged.
3. Requests SHALL go directly from the browser to the provider's API.
4. The page SHALL state the API model id used for every answer.

### Requirement 3: Mention detection
1. For each answer the system SHALL report whether the brand is mentioned (case-insensitive, also matching the bare domain), the position of the first mention among named businesses, and the other business names found.
2. The page SHALL state that an API answer is not the same as what a person sees in a consumer chat app.

### Requirement 4: Demo mode
1. WHEN no key is entered THEN the user SHALL be able to run demo mode, which replays a static JSON sample from `/demo`.
2. Demo samples SHALL only use canlah.ai or clearly fictional brands, and SHALL be labelled "Demo sample" with the date it was recorded.
