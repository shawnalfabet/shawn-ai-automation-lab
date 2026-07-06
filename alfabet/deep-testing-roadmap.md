# Deep Alfabet Playwright Testing Roadmap

A public-safe roadmap for turning this repo into a serious Alfabet E2E testing portfolio.

## Phase 1 — Foundation

- Define the lab manifesto.
- Map the test architecture layers.
- Add public-safe Playwright example specs.
- Add test-plan and PR templates.
- Document selector and flake-prevention rules.

## Phase 2 — Test architecture

Build public notes for the major layers:

1. **Smoke layer** — can the app shell and critical routes load?
2. **Navigation layer** — can users reach major sports/events/menus?
3. **State layer** — do visible states transition safely?
4. **Betslip-safe layer** — can read-only or sandbox-safe betslip behavior be validated?
5. **Observed contract layer** — can UI assertions align with network-visible state without exposing private APIs?
6. **Visual semantics layer** — can we assert meaning, not pixels?
7. **Forensics layer** — can failures produce enough evidence to debug quickly?

## Phase 3 — Creative experiments

### State-model testing

Model UI state transitions instead of writing only linear scripts.

Example states:

```text
empty betslip → selection pending → selection accepted → price changed → user confirmation required → cleared
```

### Property-based UI actions

Generate safe action sequences and assert invariants:

- no impossible disabled/enabled combinations;
- navigation never traps the user;
- loading states eventually settle;
- unavailable markets do not expose active actions.

### Synthetic fixture worlds

Create deterministic fake worlds:

- stable sports hierarchy;
- controlled market status changes;
- predictable empty/error/loaded states;
- no real customer or trading data.

### Agent-assisted test design

Use agents for planning and exploration, but commit only normal Playwright code.

```text
planner → test charter
writer → public-safe spec/harness
reviewer → selector/stability critique
healer → flake triage notes
```

## Phase 4 — Portfolio polish

- Turn README into an index of testing ideas.
- Add diagrams for test architecture.
- Add examples that read like real engineering artifacts.
- Keep every commit small, useful, and focused.

## What not to build here

- Fake green-square commits.
- Private source mirrors.
- Real credentials or env files.
- Full production test suites.
- Anything that depends on a private Alfabet deployment.
