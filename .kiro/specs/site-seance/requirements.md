# Requirements: Site Séance

## Introduction
Site Séance checks how readable a public website is for AI crawlers and AI answer engines. It needs no API key and no sign-in, and the same input always gives the same result.

## Requirements

### Requirement 1: Fetch a site safely
**User story:** As a site owner, I want to paste my URL and have the tool fetch the pages it needs, so that I do not have to upload anything.

#### Acceptance criteria
1. WHEN the user submits a URL THEN the system SHALL fetch the homepage HTML, `/robots.txt`, `/llms.txt` and `/sitemap.xml` through the fetch proxy.
2. IF the URL scheme is not `http` or `https` THEN the system SHALL reject it with a plain-language error.
3. IF the host resolves to a private, loopback, link-local, CGNAT or cloud-metadata address (before or after any redirect) THEN the proxy SHALL refuse the request.
4. The proxy SHALL use GET only, a timeout of 8 seconds, a 2 MB body cap and at most 3 redirects.
5. WHEN a resource returns 404 or times out THEN the system SHALL record that outcome as evidence and continue with the other checks.

### Requirement 2: AI crawler rules in robots.txt
**User story:** As a site owner, I want to know which AI crawlers my robots.txt blocks, so that I can decide what to allow.

#### Acceptance criteria
1. The system SHALL evaluate robots.txt for a list of AI crawler user-agent tokens. Each token in the list SHALL carry a link to the operator's own public documentation; tokens without such a link SHALL NOT be listed.
2. The matcher SHALL follow RFC 9309: user-agent group matching is case-insensitive, the longest matching path rule wins, and Allow wins a tie with Disallow.
3. WHEN no group matches a crawler THEN the system SHALL fall back to the `*` group, and if none exists treat the path as allowed.
4. The result for each crawler SHALL show allowed / blocked for `/` and the exact rule line that decided it.

### Requirement 3: llms.txt
1. WHEN `/llms.txt` exists THEN the system SHALL check it against the structure published at llmstxt.org (H1 title, optional blockquote summary, H2 sections of markdown links) and report each problem found.
2. WHEN it does not exist THEN the system SHALL report "missing" with a link to the format.

### Requirement 4: Structured data
1. The system SHALL extract every `<script type="application/ld+json">` block from the homepage, including `@graph` arrays.
2. The system SHALL list the schema.org `@type` values found and flag JSON that fails to parse.
3. The system SHALL report whether at least one of Organization, LocalBusiness (or a subtype), Product or FAQPage is present.

### Requirement 5: Content visible without JavaScript
1. The system SHALL extract visible text from the raw HTML (excluding script, style, noscript and template content) and report its word count.
2. The system SHALL report the raw number only and SHALL NOT claim what any specific AI system can or cannot see.

### Requirement 6: Basic meta
1. The system SHALL report presence and value of `<title>`, meta description, canonical link and hreflang links.

### Requirement 7: Haunting Score
1. The system SHALL compute a score from 0 to 100 using fixed weights shown on the results page.
2. The score SHALL be bounded: it is never below 0 or above 100 for any input.
3. The score SHALL be monotonic: changing any single check from fail to pass SHALL NOT lower the score.
4. Every check in the score SHALL link to its raw evidence (the fetched text or rule line).
