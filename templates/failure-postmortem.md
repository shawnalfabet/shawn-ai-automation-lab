# E2E Failure Postmortem Template for Alfabet-Style Playwright Tests

Use this template when an Alfabet-style Playwright failure deserves more than a rerun. The goal is to turn a failed E2E signal into a small, reviewable learning artifact: what user risk was protected, what evidence was observed, what root cause was found, and what prevention was added.

This template is public-safe by design. Keep examples synthetic and generalized. Do not include proprietary selectors, internal routes, credentials, environment variables, customer data, screenshots, traces, videos, HAR files, storage state, raw payloads, or private implementation details.

## Postmortem header

| Field | Notes |
|---|---|
| Test or spec | Public-safe test title or generalized area name |
| Date detected | When the failure was first observed |
| Environment | local, CI, browser project, shard, retry attempt; avoid private hostnames |
| Alfabet-style area | market navigation, event detail, betslip review, account-safe read-only flow, etc. |
| Protected risk | The user-impacting or money-impacting risk the test was meant to catch |
| Failure class | product defect, test design gap, fixture gap, environment issue, unknown |
| Current status | open, mitigated, fixed, quarantined with expiry, monitoring |

## Executive summary

Write three short sentences:

1. What failed?
2. Why did it matter to the user or business risk?
3. What changed to prevent recurrence?

```md
A synthetic changed-market scenario failed because the UI reached an inconsistent review state after a deterministic transition. This mattered because stale context must not leave a money-impacting action enabled. Prevention was added by tightening the invariant, making the fixture transition explicit, and preserving trace-on-retry evidence for the same path.
```

## Expected invariant

State the behavior the test was protecting before discussing implementation details.

```md
When [observable state or transition] occurs while [user intent] is active, the UI must [safe behavior] before [primary action] is available.
```

Examples:

- When a synthetic market changes after selection, the UI must require explicit review before continuation.
- When a market is suspended, the betslip summary and primary action must both reflect the suspended state.
- When a navigation recovery path is shown, the route, heading, and fallback action must describe the same safe state.

## Timeline

Use timestamps or ordered steps. Keep the evidence public-safe.

| Step | Observation | Evidence source | Notes |
|---|---|---|---|
| 1 | Test opened deterministic fixture world | `test.step`, fixture seed | Synthetic data only |
| 2 | User intent started | trace step or annotation | No private selectors |
| 3 | State transition occurred | fixture event or network category | Category/status only, no raw URL/body |
| 4 | Invariant failed | assertion message | Quote generalized message |
| 5 | Retry behavior observed | Playwright retry result | same seed or different path? |

## Evidence bundle

Attach or reference evidence in the private test system as appropriate, but do not commit raw artifacts to the public lab.

| Evidence | What to record | Public-safe handling |
|---|---|---|
| Test annotations | risk, fixture, seed, model path, browser project | Synthetic labels only |
| Trace | step order, actionability, snapshots | Retain in CI artifact storage; do not commit |
| Video/screenshot | visual state at failure | Use only in private triage unless sanitized synthetic data is guaranteed |
| Console summary | error category, count, first relevant message type | Redact payloads and implementation details |
| Network summary | route category, status class, duration bucket | No private URLs, headers, tokens, or bodies |
| DOM/accessibility snapshot | visible labels and roles around the failing assertion | Avoid proprietary copy or selectors when publishing notes |

## Root cause analysis

Choose the smallest explanation supported by evidence.

| Candidate class | Evidence that supports it | Evidence that rules it out | Conclusion |
|---|---|---|---|
| Product behavior defect | invariant fails with same seed and same state path | product owner says behavior changed intentionally |  |
| Test design gap | assertion checks incidental copy/layout/timing | user-visible invariant still holds |  |
| Fixture gap | synthetic world cannot force the intended transition deterministically | transition is stable and logged |  |
| Async boundary gap | UI becomes correct after assertion window | assertion waits on a real visible state |  |
| Selector/locator drift | locator matches wrong repeated row or non-unique control | role/name is unique and scoped |  |
| Environment issue | browser crash, CI pressure, outage, shard-only pattern | reproducible locally with same seed |  |

### Five whys

1. Why did the assertion fail?
2. Why was that state reachable?
3. Why did the fixture or setup allow it?
4. Why did the existing test design not isolate it sooner?
5. Why would this class recur without a prevention change?

## Impact assessment

| Question | Answer |
|---|---|
| Could a user see this state? |  |
| Could the state affect money-impacting behavior? |  |
| Was the failure deterministic with the same seed? |  |
| Did retries pass or fail consistently? |  |
| Is this a release blocker, test blocker, or monitoring signal? |  |
| Who owns the next action? |  |

## Fix and prevention plan

Separate the immediate fix from the durable prevention.

| Layer | Action | Owner | Verification |
|---|---|---|---|
| Product behavior | Fix user-visible state, disabled action, validation, or recovery copy |  | Re-run same seed and adjacent transitions |
| Test design | Assert invariant at each state boundary instead of incidental details |  | Failure message names protected risk |
| Fixture design | Add deterministic transition, seed, or recovery scenario |  | Scenario is reproducible locally and in CI |
| Observability | Add annotation, step label, console summary, or network category |  | Postmortem can be written without guessing |
| Flake control | Remove arbitrary timeout, add semantic wait, isolate setup |  | Retry pattern improves without hiding defects |

## Playwright regression checklist

Before closing the postmortem, confirm:

- [ ] The test title names the protected Alfabet-style behavior.
- [ ] `test.step` labels describe the state path that failed.
- [ ] `testInfo.annotations` include risk, fixture/scenario, and model path where useful.
- [ ] The fixture world is synthetic, deterministic, and reproducible by seed or scenario ID.
- [ ] The assertion checks a user-visible invariant, not a private implementation detail.
- [ ] The fix was verified on the original failing path and at least one adjacent path.
- [ ] Any quarantine has an owner, reason, and expiry date.
- [ ] No secrets, private selectors, internal URLs, raw payloads, traces, screenshots, videos, or storage state are committed.

## Example public-safe postmortem skeleton

```md
# Postmortem: Changed synthetic selection did not require review

## Summary

A Playwright test for a synthetic market-detail flow found that the review state was not visible after a changed-selection transition. The protected risk was stale context before a money-impacting continuation. The prevention was to add a deterministic changed-selection fixture, assert the review invariant immediately after transition, and annotate the trace with the model path.

## Invariant

When a selected synthetic market changes, the UI must require explicit review before the primary continuation action is enabled.

## Evidence

- Fixture: `synthetic-market-transition-v1`
- Seed: `postmortem-example-seed`
- Model path: `stable → selected → changed → review-required`
- Browser project: `chromium-desktop`
- Retry behavior: failed with the same seed before the fix; passed after prevention
- Network summary: market-data category returned success status; no raw endpoint recorded

## Root cause

The assertion waited for the summary panel to update but did not verify that the primary action moved to the safe review state at the same boundary.

## Prevention

- Added an explicit invariant assertion for review-required state.
- Logged fixture seed and model path in test annotations.
- Kept trace-on-retry enabled for this risk class.
```

## Related lab notes

- [Deep Test Charter Template](deep-test-charter.md)
- [Playwright Flake Forensics](../playwright/flake-forensics.md)
- [Playwright Observability Playbook](../playwright/observability-playbook.md)
- [Playwright State-Model Testing](../playwright/state-model-testing.md)
