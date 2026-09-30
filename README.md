AI Lead Intelligence & Routing

A production-style B2B lead intelligence, enrichment, verification, scoring, and routing system built with n8n, JavaScript, Apollo, Airtable, Tavily, OpenAI, and Softr.

The project demonstrates how I approach GTM / RevOps / Marketing Automation problems as systems problems: not just connecting APIs, but defining contracts between layers, validating data, handling ambiguous identities, separating recoverable failures from terminal ones, and producing evidence-backed outputs for Sales.

Live demo:
https://my-portfolio-enrichment.softr.app/

---

What the system does

A lead enters the pipeline with a business email.

The system then:

1. validates and normalizes the input;
2. verifies the email domain;
3. resolves the company behind the domain;
4. resolves the individual behind the email;
5. verifies current employment;
6. determines the relationship between the identified person and organization;
7. discovers and verifies relevant HR contacts;
8. calculates Account Fit, Person Fit, and Data Confidence;
9. assigns a routing outcome;
10. researches the company using web sources;
11. generates a grounded sales-intelligence brief;
12. persists the final result to Airtable and exposes it through the portfolio UI.

The workflow is deliberately designed to distinguish between:

- a confident automated decision;
- insufficient evidence;
- conflicting evidence;
- a transient infrastructure/provider failure;
- invalid input;
- and a case that requires human review.

---

Architecture

flowchart TD
    A[Lead submitted] --> B[Slice 1: Input Guardrails]

    B -->|valid business input| C[Parent Core Orchestrator]
    B -->|invalid / privacy / domain failure| Z[Terminal outcome]

    C --> D[2A Account Resolution]
    D --> E[2B Person Resolution]
    E --> F[2C Employer Verification]
    F --> G[2D Organization Relationship]
    G --> H[2E HR Discovery + Candidate Verification]

    H --> I[Account Fit]
    I --> J[Person Fit]
    J --> K[Data Confidence]
    K --> L[Final Routing]

    L --> M[2F Sales Intelligence]
    M --> N[Tavily research]
    N --> O[Grounded OpenAI extraction]
    O --> P[Sales Brief]

    P --> Q[Airtable persistence]
    Q --> R[Softr result UI]

    D -->|ambiguous| MR[Manual Review]
    E -->|identity conflict| MR
    F -->|employment uncertainty| MR
    G -->|relationship uncertainty| MR
    H -->|verification uncertainty| MR

The parent workflow acts as an orchestrator rather than allowing layers to call each other implicitly. Each layer receives a canonical state object, validates its expected contract, performs its work, and returns a controlled lifecycle state.

---

Repository structure

The repository contains the exported n8n workflows that implement the system.

Entry and orchestration

"Slice 1 (Webhook → Guardrails).json"

Handles the initial request lifecycle:

- Airtable trigger;
- idempotency checks;
- input normalization;
- privacy handling;
- business/personal/system inbox classification;
- email-domain verification;
- invalid-domain handling;
- transient technical failures;
- transition into the main orchestration pipeline.

"Slice 2 — Parent Core Orchestrator.json"

Coordinates the complete enrichment lifecycle.

It executes Layers 2A–2F, evaluates their output contracts, stops on manual-review or failure states, computes fit/confidence dimensions, performs final routing, protects Airtable writes against stale executions, and prepares the result returned to the UI.

---

Layer 2A — Account Resolution

"Layer 2A (Account Resolution).json"

Resolves a verified business email domain to a canonical organization.

Key responsibilities:

- strict input-contract validation;
- Apollo organization lookup;
- provider-response normalization;
- transient vs permanent error classification;
- controlled retry behavior;
- canonical company construction;
- ambiguity detection;
- manual-review routing;
- persistence and final conformance checks.

The layer does not treat a provider response as automatically trustworthy. Provider data is first converted into an internal canonical representation before downstream logic can use it.

---

Layer 2B — Person Resolution

"Layer 2B (Person Resolution).json"

Attempts to identify the individual represented by the submitted email.

The workflow includes:

- upstream contract validation;
- special handling for role inboxes;
- exact person matching;
- optional profile hydration;
- provider-observation reconciliation;
- LinkedIn/profile-anchor handling;
- web corroboration through Tavily where necessary;
- name corroboration;
- final person-resolution decision;
- controlled terminal states.

The goal is to keep identity resolution separate from assumptions about employment or company fit.

---

Layer 2C — Employer Verification

"Layer 2C (Employer Verification).json"

Determines whether the resolved person can be reliably associated with the expected employer.

The workflow includes:

- eligibility checks;
- exact-profile retrieval;
- LinkedIn URL invariants;
- profile/name corroboration;
- identity-conflict detection;
- employment evaluation;
- explicit skip, continue, manual-review, and failure states.

A resolved person and a resolved company are not automatically treated as a verified employment relationship.

---

Layer 2D — Organization Relationship

"Layer 2D (Organization Relationship).json"

Evaluates relationships between organizations when labels or domains alone are insufficient.

The workflow separates:

- obvious same-entity cases;
- label-only evidence;
- external relationship evidence;
- provider/service failures;
- verification logic;
- canonical relationship outcomes.

This layer exists to avoid collapsing different organizations into a single account simply because their names or other superficial attributes appear related.

---

Layer 2E — HR Discovery + Candidate Verification

"Layer 2E — HR Discovery + Candidate Verification.json"

This is one of the deeper parts of the system.

It discovers potential HR/recruiting contacts and then verifies and ranks them rather than blindly returning provider search results.

The workflow includes:

- discovery-result normalization;
- an identity graph for duplicate/repeated observations;
- component reconciliation;
- inbound comparison;
- title classification;
- verification-pool selection;
- profile hydration;
- exact-profile retrieval;
- employment-currentness evaluation;
- target-organization comparison;
- final HR-role classification;
- verified-candidate ranking;
- evidence generation;
- invariant assertions;
- canonical output generation.

Provider sub-workflows

The provider interaction is isolated into dedicated workflows:

- "2E Provider — Apollo HR Discovery v1 (LIVE).json"
- "2E Provider — Apollo Hydration v1 (LIVE).json"
- "2E Provider — Apollo Exact Profile v1 (LIVE).json"

Each provider workflow owns its HTTP lifecycle and retry behavior instead of leaking provider-specific implementation details into the business-logic layer.

---

Layer 2F — Sales Intelligence

"Layer 2F - Sales Intelligence - Tavily Batch Extract + Native OpenAI.json"

The final enrichment stage converts validated account and routing context into actionable sales intelligence.

It combines Tavily research with OpenAI extraction/synthesis, while keeping the research pipeline bounded by explicit contracts and provenance.

Key implementation elements include:

- parent routing gate;
- input-envelope validation;
- input fingerprint verification;
- frozen execution configuration;
- canonical search-query generation;
- search-result normalization;
- source selection;
- batched Tavily extraction;
- canonical source identity;
- provenance tracking;
- grounded observation construction;
- research-state reduction;
- deterministic signal ranking;
- company-context assembly;
- sales-brief generation;
- terminal execution-state resolution.

The workflow also contains JavaScript implementations for deterministic hashing and canonicalization used in execution integrity and idempotency logic.

Routing outcomes determine whether Sales Intelligence should run at all. Examples include:

- "SALES_HIGH_PRIORITY"
- "SALES_STANDARD"
- "ACCOUNT_OPPORTUNITY"
- "NURTURE"

Cases such as invalid input, failed enrichment, privacy stops, manual review, or disqualified fit do not unnecessarily invoke the research/LLM pipeline.

---

Reliability and failure handling

A major goal of this project was to design the automation as a stateful system, rather than a linear chain of API calls.

Contract validation

Layers validate their expected input and output state before allowing processing to continue.

Unexpected combinations are treated as contract violations rather than silently propagated downstream.

Transient-only retries

API requests distinguish between failures that may succeed on retry and failures that should terminate immediately.

Examples of retryable conditions include:

- timeouts;
- connection failures;
- HTTP 408;
- HTTP 429;
- selected HTTP 5xx responses.

A dedicated Airtable retry workflow follows the same principle.

Manual review as a first-class state

Ambiguity is not automatically converted into false certainty.

Identity conflicts, unresolved employer relationships, or insufficient evidence can intentionally produce a "MANUAL_REVIEW" outcome.

Idempotency

The pipeline tracks execution/run state so duplicate or stale executions do not blindly overwrite current results.

Provider abstraction

External providers are adapted into canonical internal structures before their data reaches decision logic.

This limits provider-specific assumptions and makes downstream reasoning more deterministic.

Evidence before AI

The LLM is used downstream of deterministic validation and research collection.

It does not decide whether an email domain exists, whether two identifiers match, or whether a provider request succeeded.

---

Technology

Area| Technology
Workflow orchestration| n8n
Workflow logic| JavaScript
Account & person enrichment| Apollo
Web research| Tavily
AI extraction / synthesis| OpenAI
Operational data store| Airtable
Portfolio / result interface| Softr
Domain validation| DNS / mail-domain verification
Integration style| REST APIs, sub-workflows, canonical JSON contracts

---

Engineering principles demonstrated

This project was primarily built to explore how GTM automation can be engineered as a reliable system rather than a collection of disconnected automations.

The implementation demonstrates:

- workflow orchestration;
- API integration;
- JavaScript inside automation infrastructure;
- deterministic data normalization;
- state-machine-style routing;
- retry and failure policy;
- canonical data contracts;
- identity resolution;
- reconciliation of conflicting provider evidence;
- idempotency;
- human-in-the-loop routing;
- web research;
- LLM integration with source grounding;
- sales-intelligence generation;
- operational persistence.

---

Suggested review path

If you are reviewing this repository for technical depth, I recommend starting with:

1. "Slice 2 — Parent Core Orchestrator.json"
For the overall system architecture and lifecycle.

2. "Layer 2E — HR Discovery + Candidate Verification.json"
For identity reconciliation, verification logic, ranking, and invariant checks.

3. "Layer 2F - Sales Intelligence - Tavily Batch Extract + Native OpenAI.json"
For the research/AI pipeline, canonicalization, provenance, deterministic processing, and execution-integrity logic.

4. "Layer 2B (Person Resolution).json"
For identity resolution, provider reconciliation, retry handling, and external corroboration.

---

About this repository

This is a portfolio implementation, published to make the technical depth of the project inspectable.

The repository contains n8n workflow exports and embedded JavaScript logic. It is intended primarily for architecture and implementation review rather than one-click deployment: external services, credentials, environment variables, Airtable schemas, and n8n workflow references must be configured separately for another environment.
No API secret values should be committed to this repository.

---

Project links

Live demo
https://my-portfolio-enrichment.softr.app/

GitHub repository
https://github.com/Zamotirina/ai-lead-intelligence-routing

---

Built as a portfolio project at the intersection of GTM Engineering, Marketing/Revenue Operations, CRM Automation, and software systems design.
