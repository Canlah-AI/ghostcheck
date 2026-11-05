# Requirements: Haunted UI

## Introduction
The interface for the Costume Contest category: a haunted house where your brand is a ghost that becomes visible as your site becomes readable to AI.

## Requirements

### Requirement 1: Ghost visibility
1. The ghost's opacity SHALL map to the Haunting Score (low score = faint ghost).
2. WHEN a check passes THEN the ghost SHALL visibly gain form, and the passing check SHALL be named on screen.

### Requirement 2: Readable and accessible
1. All text SHALL meet WCAG 2.1 AA contrast against its background, including over fog effects.
2. WHEN `prefers-reduced-motion` is set THEN the system SHALL disable fog drift and ghost animation and show static states.
3. Every result SHALL be reachable by keyboard and announced to screen readers.

### Requirement 3: No third-party marks
1. The UI SHALL NOT display third-party logos or trademarks. AI providers are shown as text labels plus API model id.

### Requirement 4: Judge path
1. A first-time visitor SHALL be able to run Site Séance on a sample URL and open demo mode within two clicks, without signing in or entering a key.
