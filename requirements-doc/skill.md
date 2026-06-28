---
name: requirements-doc
description: Write a requirements document for a feature — structured dialogue to elicit contracts, UI specs, and behavioral requirements, then produce a formal document with a review checklist.
disable-model-invocation: true
---

# Requirements Document

Guide the user through a structured conversation to produce a requirements document,
then generate a review checklist against it.

The output is a markdown document meant for copying into Confluence. It specifies
what is most expensive to change: contracts between components, data schemas,
behavioral commitments, and UI boundaries.

## Phase 1: Requirements Elicitation

Ask the user about each dimension below, **one question at a time**. Prefer
multiple-choice or yes/no questions when possible. Confirm the user has said
enough before moving to the next dimension. Skip dimensions the user says
don't apply.

### 1. Feature Scope

"What is this feature? What problem does it solve?"

Follow up:
- "Is there anything explicitly out of scope?"
- "Who are the users or actors?"

### 2. Components & Boundaries

"Which components or systems are affected?"

Guide the user by naming possibilities:
- Database
- API server
- UI
- Message queue / event bus
- External service integration

"Are there dependencies between components — does one need changes before another can work?"

### 3. Anchor Identification

"Looking at the affected components, what anchors these requirements?"

Present the two scenarios:

| Anchor | Description |
|--------|-------------|
| **UI-driven** | The user experience and screen designs are the primary driver. API contracts and data model serve the UI. |
| **Backend-constrained** | Existing data models, legacy schemas, or service contracts constrain what the UI can do. |

If the user says "both", treat it as backend-constrained — the harder constraints determine the solution space.

Then follow the visitation order below, visiting **only the sections that are relevant** based on Components & Boundaries. If no UI changes are involved, skip UI Screens and Cross-Cutting UI Patterns. If no data model changes, skip Data Model, and so on. Each section below has a "(If … relevant)" guard — respect it.

Always explore from the anchor outward.

**Visitation Order**

| Step | UI-Driven | Backend-Constrained |
|------|-----------|---------------------|
| 1st  | §4 UI Screens | §7 Data Model |
| 2nd  | §5 Cross-Cutting UI Patterns | §6 API Contracts |
| 3rd  | §6 API Contracts | §8 Message/Event Contracts |
| 4th  | §7 Data Model | §4 UI Screens |
| 5th  | §8 Message/Event Contracts | §5 Cross-Cutting UI Patterns |

Auth (§9), Edge Cases (§10), and Non-Functional (§11) always come last regardless of anchor.

Use these transition phrases as you move from the anchor to the dependent sections:

- **UI → API:** "Now that we know what the UI needs, let's define the API contracts to support it."
- **API → Data:** "Given these API contracts, what data must be stored?"
- **Data → API:** "Given this data model, what endpoints does it expose?"
- **API → UI:** "Given these API contracts, what UI can we build on them?"

### 4. UI Screens

(If UI changes are relevant)

"What pages or screens are being added or modified?"

For each screen, establish:
- Route path and what it displays
- What data it needs (reference §6 endpoints)
- What user actions are available
- What states are needed (loading, empty, error, populated)

Also ask about navigation:
- "Does this affect the nav structure — new routes, new groups, changed labels?"

### 5. Cross-Cutting UI Patterns

Ask about each pattern. Do NOT assume defaults. The user may:
- Describe the pattern from scratch
- Refer to an existing spec or Confluence page ("same as the trading dashboard")
- Say it's not applicable

**Tables:** "How should tables behave — loading state, empty state, error state, row hover, sorting, responsive behavior?"

**Modals:** "What close behavior (X, Escape, click-outside)? Backdrop? Body scroll? Transitions?"

**Forms:** "How should validation work — inline below fields, tooltip, or summary? Submit button behavior? Server error mapping?"

**Notifications:** "Where positioned? Auto-dismiss timing? Visual treatment for success vs error vs info? Retry actions?"

**Error Handling:** "What's the difference between ephemeral and persistent errors? When does an error become critical (full-page vs inline)?"

### 6. API Contracts

(If API changes are relevant)

"What endpoints or RPC methods are new or modified?"

For each endpoint, establish:
- Method, path, and description
- Request parameters (path, query, body)
- Response shape
- Error codes and what triggers each
- Validation rules
- Auth requirements (if any)

### 7. Data Model

(If DB changes are relevant)

"What new tables, collections, or fields are needed?"

For each entity, ask:
- What fields, types, and constraints?
- What are the primary keys and indexes?
- Which fields are required vs optional?
- What are the defaults, if any?
- Any migration considerations for existing data?

### 8. Message / Event Contracts

(If messaging or async events are relevant)

"What events or messages flow between components?"

For each event, establish:
- Event name and payload fields
- Producer and consumer(s)
- Routing (queue, topic, routing key)
- Delivery guarantees (at-least-once, exactly-once, best-effort)
- Ordering requirements, if any

### 9. Auth & Security

If the agent senses auth is relevant — otherwise skip.

"Is there any auth or security consideration?" If yes:
- Auth method (session, JWT, API key, OAuth)
- Which endpoints/pages require auth
- What happens on session expiry
- Any permission differences between users
- CSRF or other protections

### 10. Edge Cases

"What happens when things go wrong?"

Prompt with specific scenarios:
- Network timeout during an API call
- Concurrent edits to the same resource
- Partial success (some items updated, some failed)
- Invalid or malicious input
- Backend returns an unexpected response

### 11. Non-Functional

(Optional dimension) "Any performance, monitoring, or alerting requirements?"

## Phase 2: Document Generation

Write the requirements document using the structure in `TEMPLATE.md`.

- Every applicable section is populated from the elicitation answers
- The document is self-contained — anyone reading it understands the requirements
- No implementation details — only what is committed to
- Use table formats for contracts (fields, types, constraints)
- Use consistent field names across DB, API, and UI sections

Offer to either write the document inline or to a file. Output clean markdown.

## Phase 3: Review Checklist

After writing the document, run each check against it. For each check, cite
section references from the document (e.g., "§3.2 POST /api/orders → §4.3 OrderListPage").

1. **API coverage** — Every endpoint in the Contracts section has at least one consumer in the UI or Behavioral sections.

2. **Error handling coverage** — Every error code or failure mode listed in Contracts has a corresponding handling path in the UI or Behavioral sections.

3. **Required field validation** — Every field marked "required" in the Data Model has a validation rule in the API section.

4. **UI state coverage** — Every screen lists its loading state, empty state, and error state.

5. **Field consistency** — Field names are spelled and typed consistently across DB schema, API contracts, and UI sections.

6. **Completion paths** — Every user action has both a success path and an error path.

7. **Glossary coverage** — Every domain term used in the document is defined in §2 (Glossary) or the project CONTEXT.md.

For each check: ✅ PASS or ❌ FAIL with reasoning.

Present results to the user. If any check fails, offer to fix the document.
