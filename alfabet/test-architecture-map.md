# Alfabet Playwright Test Architecture Map

This map describes a public-safe layered E2E architecture for Alfabet-style betting and trading interfaces. It is intentionally product-shaped without depending on private URLs, selectors, screenshots, credentials, APIs, or implementation details.

The goal is to make the suite explain **which risk each test layer owns** so Playwright does not become one large bucket of slow, flaky browser scripts.

## Architecture principles

1. **Layer by risk, not by page.** A market card, event page, and betslip can participate in several layers depending on the behavior under test.
2. **Keep lower layers fast and boring.** Smoke and navigation tests should be cheap enough to run often.
3. **Reserve deep browser coverage for money-impacting states.** E2E is most valuable where visible UI state can affect user trust or financial decisions.
4. **Treat real-time change as normal.** Odds, status, availability, loading, and recovery are part of the model, not accidental flake.
5. **Attach evidence to failure.** A failing test should leave enough trace, console, network, screenshot, and annotation context for a reviewer to classify it quickly.

## Layer map

| Layer | Primary question | Example public-safe checks | Should avoid |
|---|---|---|---|
| Smoke | Does the app shell boot and expose the critical entry points? | Landing route loads, primary navigation is visible, unauthenticated state is understandable | Deep assertions, live data assumptions, write flows |
| Navigation | Can a user reach important product areas across viewports? | Sport list opens, event detail route is reachable, back/forward behavior preserves orientation | Private route names, brittle menu order assumptions |
| State-model | Do UI states transition safely under changing market conditions? | Available → suspended → reopened; empty betslip → selected → validation required → cleared | Assuming every state is reachable in live production data |
| Contract-observed UI | Does visible UI agree with observed network-level facts? | If an observed response marks a market unavailable, the action affordance is disabled or explanatory | Exposing private endpoints, schemas, tokens, or raw payloads |
| Visual semantics | Does the screen communicate the right meaning to the user? | Disabled action looks disabled, stale-price warning is distinguishable, loading skeleton does not imply availability | Pixel-perfect assertions for dynamic content |
| Flake forensics | Can failures be classified and fixed instead of retried forever? | Attach trace/video on retry, classify timeout vs selector drift vs data freshness vs app regression | Blind `waitForTimeout`, automatic reruns with no investigation |
| Agent-generated plans | Can AI help design coverage while committed code stays reviewable? | Agent drafts charters, invariants, and edge-case lists; humans commit normal Playwright specs | Runtime AI dependency in CI, unreviewed generated selectors |

## Recommended suite shape

```text
pre-merge
├─ smoke: app shell + critical navigation
├─ deterministic fixtures: synthetic market/event worlds
└─ lint/static checks: test conventions and selector hygiene

scheduled
├─ state-model flows against controlled test worlds
├─ cross-viewport navigation and visual semantics
├─ observed-contract UI correlation
└─ flake-forensics report generation

manual/research
├─ exploratory deep-test charters
├─ agent-assisted plan generation
└─ new harness experiments before promotion to CI
```

## Layer details

### 1. Smoke layer

Smoke tests prove that the browser can load the product and that the user is not blocked at the first meaningful screen.

Good smoke assertions are intentionally shallow:

- the shell renders without a fatal error boundary;
- primary navigation or an equivalent fallback is visible;
- unauthenticated, empty, or maintenance states communicate a clear next step;
- console errors are captured and reviewed, not ignored.

A smoke test should fail loudly when the product is unusable, but it should not encode live market availability or exact content ordering.

### 2. Navigation layer

Navigation tests protect the user's ability to move through the product model: sport, competition, event, market group, account boundary, and betslip surface.

Useful navigation checks:

- route changes preserve a recognizable page heading or selected context;
- mobile navigation exposes the same critical areas through an appropriate interaction model;
- refresh and back/forward do not trap the user in a loading or empty state;
- deep links degrade safely when the target event or market is unavailable.

Public-safe examples should use generic names such as `sport`, `event`, `market`, and `selection`, not private route structures.

### 3. State-model layer

State-model tests represent the product as transitions instead of single linear scripts.

Example model:

```text
market available
  → selection added
  → price changed
  → confirmation required
  → user accepts or clears
  → betslip returns to safe state
```

The value is not only broader path coverage. The value is the explicit invariant list:

- unavailable markets never expose an enabled action;
- stale selections require review before proceeding;
- clearing the betslip removes validation warnings tied to prior selections;
- loading and recovery states eventually settle or explain why they cannot.

### 4. Contract-observed UI layer

This layer correlates what the browser observes over the network with what the user can see. It should not publish private endpoints or payloads.

Public-safe pattern:

```text
observe: response category = market unavailable
assert: UI category = action disabled with explanatory status
```

The test can store only generalized categories in its evidence, for example `available`, `suspended`, `price_changed`, or `unavailable`. That keeps the artifact useful without leaking internal schemas.

### 5. Visual semantics layer

Visual testing should focus on meaning rather than brittle screenshots. For an Alfabet-style UI, the important questions are often semantic:

- Can the user distinguish available, suspended, selected, and changed-price states?
- Does a disabled action look and behave disabled across themes and viewports?
- Are risk warnings visible near the action they affect?
- Does responsive layout preserve the relationship between market, selection, and betslip?

Screenshots can support investigation, but committed examples should avoid private product imagery.

### 6. Flake forensics layer

Flake forensics is a first-class architecture layer because real-time products create multiple valid sources of nondeterminism.

Minimum failure evidence:

- Playwright trace on retry;
- console errors and page errors;
- failed request summary without secrets;
- test annotations for scenario, data mode, viewport, and risk level;
- a short failure classification after triage.

A retry that turns red into green without producing learning should be treated as a debt signal.

### 7. Agent-generated plan layer

Agents are useful for generating candidate charters, invariants, and edge-case paths. They should not make CI mysterious.

Safe workflow:

```text
agent proposes coverage → human reviews risk and public safety → normal Playwright test or doc is committed
```

Rules for this repo:

- no runtime AI dependency in committed tests;
- no private selectors or copied implementation details;
- generated plans must be converted into readable engineering artifacts;
- review should ask whether the scenario protects a meaningful product risk.

## Promotion checklist

Before promoting a test idea into a stronger CI lane, ask:

- Which layer owns this behavior?
- What risk does it reduce?
- Does it depend on live data that should be made synthetic?
- What evidence will a failure leave?
- Can the selector strategy survive normal UI refactors?
- Is the example public-safe and free of private Alfabet details?

## Related lab notes

- [Testing lab manifesto](testing-lab-manifesto.md)
- [Deep testing roadmap](deep-testing-roadmap.md)
