---
name: backbone-plan
description: Use when the user wants a backbone plan — tests interleaved with implementation, verification leading every step. Use when converting requirements into a test-led implementation sequence.
---

# Backbone Plan

## Overview

Produce an implementation plan where tests form the **backbone**. Every step includes its verify command and expected output. Verification is first-class. **Fidelity** — how faithfully a test double mirrors the real system — is a required property, not a nice-to-have.

**Announce at start:** "I'm using the backbone-plan skill to create the implementation plan."

## The Interview

Ask one question at a time. Prefer multiple choice.

### 1. Outermost interface

What is the widest testable surface? CLI binary, HTTP endpoint, library public API. Backbone tests default to this surface — only reach internals with a justification in Notes (e.g., streaming systems that need a staged fake at an internal seam to stay deterministic).

### 2. External dependencies

First enumerate every external dependency in the requirements explicitly — list them in your output before asking about any. A missed dependency here is an untested seam in the plan.

### 2b. Credentials Audit

For each dependency that interacts with a third-party system, determine what access exists **before** choosing a simulation technique. Access falls into three tiers:

1. **Available now (tier 1)** — A credential exists and can be used by the access-gate diagnostic. Enables dual execution and live traffic recording.
2. **Available historically (tier 2)** — The team has pre-recorded fixtures or a teammate with access can record them. Enables record/replay but not dual execution.
3. **Unavailable (tier 3)** — No live access, no fixtures. Only contract stubs are feasible.

Ask: "For <dependency>, what tier of access exists?"

Record each answer. If no access exists (tier 3), runtime proofs are not possible — this must be documented as residual fidelity risk in the plan.

**Isolation.** For tier 1 and tier 2 dependencies, how will the plan reference the credential without the agent ever reading its value? Acceptable: env var name (`AWS_PROFILE`, `UPSTREAM_TOKEN`, `ANTHROPIC_API_KEY`), a tool alias (`make check-access`), or a config key the agent passes through to a subprocess it does not read. Forbidden: hardcoded secrets, `echo $TOKEN` in the plan, credential values in the agent's output.

### 2c. Simulation Techniques

For each dependency, choose a technique **constrained by the access tier from 2b**:

| Access tier | Feasible techniques (ranked) |
|---|---|
| Tier 1 (available now) | Fake → Record/replay → Contract stub → Live sandbox |
| Tier 2 (historical) | Record/replay → Contract stub |
| Tier 3 (unavailable) | Contract stub only |

- **Fake** — In-process server on the wire protocol. Deterministic, fast. Requires tier 1 access for the runtime proof.
- **Record/replay** — Capture real responses, replay. Requires at least tier 2 access (historical fixtures).
- **Contract stub** — Hardcoded responses. No live access required. Fastest to build, highest drift risk.
- **Live sandbox** — Real instance, test creds. Nondeterministic — smoke tests only, flagged optional in the plan.

Record each choice. Every backbone test must be deterministic — no live network, no human in the loop. Record/replay and contract stubs satisfy this; live sandbox does not and is reserved for optional smoke tasks.

### 2d. Fidelity Contract

**Fidelity** has four axes a simulation must mirror: **shape** (request/response structure, wire protocol), **semantics** (field meanings, side effects), **error modes** (status codes, timeouts, retry-after), and **timing/ordering** (backpressure, ordering, eventual-consistency windows). A substitute diverging on an axis relevant to the behavior under test yields a test that passes against the double but fails against the real system. The plan must document which axes each substitute covers and which it does not.

**Shape proof (mandatory for all techniques).** Derive the substitute from, or verify it against, a machine-readable source: OpenAPI spec, protobuf definition, JSON schema, or recorded HAR. Commit the source URL **and service version** (e.g., `service v3.2.1`, `openapi.yml@<commit-sha>`) in the plan.

When no machine-readable source exists, record a HAR capture of representative traffic as the shape proof. Note in the plan that surface area is partial — undocumented endpoints and error modes are fidelity gaps.

**Runtime proof (tiered by access).** Required for fakes and record/replay. Stubs are exempt but must note the gap.

*Tier 1 (access now):* At least one of:

1. **Dual execution** — A test that runs the same scenario against *both* the fake and a live sandbox (or recorded replay), asserting they agree after normalization. Flag as optional (needs creds) but its *existence* is mandatory in the plan when building a fake.
2. **Recorded scenario replay** — Capture real service traffic once, replay it through the fake, assert identical handling after normalization. Check fixtures into version control with a header: source URL, service version, capture date.

*Tier 2 (historical fixtures):* Recorded scenario replay only. The plan must note that new scenarios cannot be validated against live service without obtaining access.

*Tier 3 (no access):* No runtime proof is possible. The plan must record this as a **documented fidelity gap**. The substitute is a contract stub; it requires shape proof + citation + drift trigger. Estimate residual risk for behaviors that depend on timing, error modes, or undocumented surface area.

**Normalization protocol.** Before comparing fake output to live output, apply these normalizations (implement as a shared function, not per-test string replacement):

1. **Timestamps** — Replace ISO-8601 and Unix timestamps with a placeholder (`<TIMESTAMP>`).
2. **UUIDs / ULIDs / random IDs** — Replace with `<UUID>`, `<ULID>`, or `<ID>`.
3. **Opaque tokens / secrets** — Replace with `<TOKEN>`.
4. **Field ordering** — Sort object keys alphabetically before comparing JSON.
5. **Whitespace** — Canonicalize (minify then reformat with fixed indent).
6. **Variable-precision numerics** — Normalize floating-point to a fixed decimal precision.
7. **Optional vs null vs omitted** — Agree on a canonical representation for absent fields.

**Drift trigger.** Every source carries a **drift trigger**: when the cited service version or spec commit is bumped, reconform (re-run dual execution or re-record fixtures) before treating the simulation as valid. The plan must document how version bumps are detected (e.g., CI check comparing fixture headers against the current service version endpoint).

**Escalation.** If a dependency is critical to correctness but no runtime proof is feasible, flag it as a release gate requiring a manual smoke test against live service before deploy.

### 3. First observable behavior

"What is the first thing the system does that a user or caller can see?"

Becomes the first backbone pair. Not "parse config" — "prints startup summary" or "returns 200 on GET /health."

### 4. Validate the plan

Before drafting, check that every external dependency has all of: (a) **access tier** recorded, (b) **simulation technique** appropriate for that tier, (c) fidelity **shape proof** (source + service version), (d) fidelity **runtime proof** appropriate for the access tier (dual execution, replay, or documented gap), (e) **determinism mechanism** (pinned time, OS-assigned ports, isolated FS, no network), (f) for tier 1 and tier 2 dependencies, **credential identified** and **isolation mechanism** recorded. If any dependency is missing one, loop back to Step 2.

## Plan Structure

Numbered steps. **Odd = test. Even = implementation.** Each pair proves one observable behavior.

The plan begins with **Task 0 — Access Gate**: a fast, read-only diagnostic that verifies the agent can reach every live dependency and reports the API shape (version, endpoints reached, a known response shape). Run it first. If it fails, stop — do not proceed to backbone tests. The diagnostic references credentials by name only, never by value.

```markdown
### Task 0: Access Gate

**Step 0 — Diagnostic:**
Run `[command]` (e.g., `make check-access` or `npx tsx scripts/access-gate.ts`).
Expected: exits 0, prints each dependency name, service version, and a sample shape.

Skip if no live dependencies; otherwise mandatory before Task 1.
```

For backbone pairs:

```markdown
### Task N: [Behavior]

**Step N — Backbone test:**
```language
// Test at outermost interface
```

**Verify:** Run `[command]`. Expected: FAIL — [reason].

**Step N+1 — Implement:**
```language
// Minimal code to pass
```
Files: `path/to/file`. Notes: [key decisions, no speculation].

**Verify:** Run `[command]`. Expected: PASS. Commit.

**Commit cadence:** commit the failing test as RED (one commit), then the implementation as GREEN (next commit) — or fold both into one atomic commit. State which convention the plan uses; do not mix.

If a step introduces a fake, its Notes must cite the fidelity contract — the spec source, service version, and which runtime proof covers it (e.g. "Shape: `openapi.yml §4.2 @ abc123`; runtime: dual exec against sandbox v3.2.1").

## Credential Isolation

No secret appears in the plan, in the agent's output, or in any committed file. The plan references credentials by env var name or tool alias only — the agent never reads the value. The access-gate diagnostic reads credentials from outside the workspace (a `~/.env` file, a keychain, an env var set by the user's shell profile). The agent merely invokes the diagnostic and reads its output; the credential value is never in the agent's context. If the agent accidentally emits a secret (e.g., in an error trace), the user must rotate it.

## Writing Backbone Tests

**Start from behavior, not code.** Test name describes what a user sees: "prints startup summary" not "calls enumerateSources".

**One outcome per test.** If "and" appears in the description, split into two backbone pairs.

**State the expected failure.** The failure reason proves the test catches the right bug: "exits 1 because the upstream URL is unset" not "test fails".

**Deterministic.** Pin time (`jest.useFakeTimers`, `TZ=UTC`), use fakes for external services, isolate filesystem. Same output every run.

## Testing Side Effects

For each dependency, the backbone test uses the substitute chosen in the interview. Patterns for fakes, record/replay, stubs, and sandboxes: see [side-effects.md](side-effects.md).

If a behavior requires live integration that cannot be made deterministic, split it: test the deterministic part with a fake or stub inline, and flag the live integration as an optional smoke task. The smoke task doubles as the runtime proof (dual execution) for the deterministic substitute when live access is available.

## Verification Discipline

A step is done only when its verify command has run and matched the expected output. No exceptions.

## After the Plan

Save to `docs/plans/YYYY-MM-DD-<feature-name>.md`.

**Execution:** When the plan is approved, hand it off to a new agent for execution. Use the `handoff` skill to compact the plan into a handoff document.
