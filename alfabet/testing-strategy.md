# Alfabet Playwright testing strategy

A public-safe outline for building small, reviewable Playwright E2E coverage around Alfabet user flows.

## Goals

- Catch obvious navigation and rendering regressions early.
- Keep tests readable and close to real user behavior.
- Prefer read-only or low-risk flows unless a test environment explicitly supports writes.
- Make every generated test reviewable as normal Playwright code.

## Coverage buckets

### 1. Smoke and shell loading

Verify that the application shell loads, core navigation is visible, and the page reaches a stable state.

Examples:

- mobile app shell renders;
- header is visible;
- primary navigation opens;
- sports/event container appears.

### 2. Navigation and menu behavior

Focus on stable interactions that do not mutate business data.

Examples:

- hamburger menu opens and closes;
- footer/menu links are visible;
- sports navigation exposes expected sections;
- active tab state changes after navigation.

### 3. Betslip-safe read-only checks

Where allowed, validate only safe UI states unless a dedicated sandbox write flow exists.

Examples:

- betslip container is present;
- empty state is clear;
- validation messages are visible when no selection exists;
- no real bet placement unless explicitly authorized in test env.

### 4. API response sanity

Use network waits only when they make the UI state less flaky.

Examples:

- wait for initial data endpoint before asserting populated UI;
- assert UI fallback when a response is empty;
- avoid coupling tests to private endpoint names in public examples.

## Test quality rules

- Use role, label, and `data-testid` locators before brittle CSS selectors.
- Avoid `waitForTimeout`; prefer locator assertions and network/UI state waits.
- Keep each test focused on one user-visible behavior.
- Add comments explaining source evidence and intent.
- CI should run normal Playwright tests with no AI runtime dependency.

## Public repo safety

This public repo should contain generalized examples only. Do not include internal URLs, credentials, private selectors, screenshots, traces, or proprietary implementation details.
