# Document Information

- **Title:** impl-plan.md — Advanced Accommodation Search Filters
- **Status:** Draft for human review (planning only)
- **Repo:** https://github.com/saketbadame09/CaptstoneCodeMei
- **Target branch:** `feature/advanced-accommodation-filters-requirements`
- **PR branch:** `feature/advanced-accommodation-filters-impl-plan`
- **Last updated:** 2026-11-27

---

# Planning Objectives

1. Translate approved requirements into an actionable, dependency-aware implementation plan.
2. Ensure backward compatibility: existing search works unchanged when new filters are omitted.
3. Include validation, error handling, logging/observability, security/abuse guardrails, and a test strategy.
4. Maintain traceability from implementation tasks to requirements/architecture/design-review.
5. Avoid inventing technology choices; keep undecided architecture topics as open questions.

---

# Source Documents

Ordered by authority as applied in this repo:

1. **Approved requirements:** `requirement.md` (repo uses singular filename)
2. **Approved architecture:** `docs/Architecture.md` (explicitly marked DRAFT and contains open questions)
3. **Approved design review:** `docs/design-review.md` (Approved with conditions; captures key decisions)
4. **Conversation decisions:**
   - Use repo/branch `feature/advanced-accommodation-filters-requirements` as the base.

---

# Implementation Principles

1. **Backward compatible API**: new query params optional; omission yields existing behavior.
2. **Deterministic semantics**:
   - AND across categories (existing filters ∧ amenities ∧ propertyType).
   - **Amenities multi-select = ALL (AND)** within amenities (design-review decision).
   - **propertyType = single-select**; multi-value is **HTTP 400** (design-review decision).
3. **Edge validation**: allowlist enums; reject invalid values early with 400.
4. **Layer separation** per architecture intent: controller/transport ↔ application/use-case ↔ domain ↔ infra/query.
5. **Observability**: structured logs for request/validation failures (no PII); metrics-ready.
6. **Performance guardrails**: enforce pagination bounds and reasonable filter size limits.

---

# Technical Work Breakdown Structure

**Epic:** Advanced Filters in Accommodation Search

1. Contract & validation
2. Domain model (enums/criteria)
3. Application use-case updates
4. Infrastructure query/repository updates (backend-dependent)
5. Data model/migrations (if needed)
6. Tests (unit/integration/contract/non-functional)
7. Documentation & CI verification

---

# Implementation Phases

> Important: This branch currently contains only documentation files (no application code). Phase 0 includes discovery to locate/merge the actual service code before concrete file-level work can proceed.

- **Phase 0 (P0):** Repo/service discovery + decision closure
- **Phase 1 (P0):** API contract + domain model + validation (backend-agnostic)
- **Phase 2 (P0/P1):** Infra query + repository + endpoint wiring (backend-specific)
- **Phase 3 (P1):** Test expansion, observability hardening, documentation, CI checks

---

# Task Details

## PH0-T01 — Repository discovery & baseline mapping

- **Phase:** 0
- **Priority:** P0
- **Dependencies:** None
- **Relevant requirement refs:** `requirement.md` (no explicit IDs)
- **Relevant architecture:** `docs/Architecture.md` §4–§5 (layering; backend undecided)
- **Description:**
  - Identify where the accommodation search endpoint is implemented (controller/route).
  - Identify the current search backend (SQL DB vs search engine vs external vendor API).
  - Identify how amenities/propertyType data is represented today (columns, join tables, JSON, etc.).
  - Identify existing pagination/sorting behavior and error handling patterns.
- **Expected files/components (to inspect):**
  - API controller/routes for accommodation search
  - DTO/request parsing layer
  - Search use-case/service
  - Repository/query builder
  - DB schema/migrations folder (if any)
- **Implementation notes:**
  - If code is not present in this repo/branch, determine source branch/repo to merge from.
- **Acceptance criteria:**
  - A short “current-state map” is added to this plan (or linked doc) listing key modules and backend choice.
- **Verification method:** Manual repo inspection + (if available) run build/test commands.
- **Estimated effort:** 2–4 hours (assumes code exists somewhere accessible).

## PH0-T02 — Close API contract decisions (query param encoding, pagination)

- **Phase:** 0
- **Priority:** P0
- **Dependencies:** PH0-T01
- **Relevant requirement refs:** `requirement.md` acceptance criteria (AC1–AC7 in text)
- **Relevant architecture/design:** design-review notes on API underspecification
- **Description:** Decide and document:
  - Amenities encoding: repeated param (e.g., `amenities=WIFI&amenities=BREAKFAST`) vs CSV.
  - `propertyType` single value only; define error response for multi-value.
  - Pagination: `page`/`pageSize` (or `offset`/`limit`), defaults, max pageSize.
  - Response shape: include `total` or not.
  - Sorting defaults if applicable.
- **Expected files/components:** `impl-plan.md` “API Contract” section (or add `docs/api-contract.md`).
- **Acceptance criteria:** API owner/lead approves contract decisions.
- **Verification method:** Human approval comment in PR.
- **Estimated effort:** 1–2 hours.

## PH0-T03 — Close backend decision & query approach

- **Phase:** 0
- **Priority:** P0
- **Dependencies:** PH0-T01
- **Relevant requirement refs:** `requirement.md`
- **Relevant architecture:** `docs/Architecture.md` §5; design-review backend ambiguity
- **Description:** Confirm whether filtering is implemented via:
  - A) SQL DB query changes, or
  - B) search engine query changes, or
  - C) external vendor API parameter mapping.
- **Expected files/components:** Decision recorded in plan.
- **Acceptance criteria:** Backend approach explicitly approved.
- **Verification method:** Human approval.
- **Estimated effort:** 0.5–1 hour.

## PH1-T01 — Define filter domain enums & criteria model

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** PH0-T02
- **Relevant requirement refs:** Amenities + property type filters
- **Relevant architecture:** layered separation; extensible criteria model
- **Description:**
  - Define allowlisted enum values for Amenities: Free Wi‑Fi, Breakfast included.
  - Define allowlisted enum values for PropertyType: Hotel, Villa.
  - Define a criteria object that composes with existing criteria.
  - Encode semantics (AND across categories; amenities ALL/AND) in domain/application layer.
- **Expected files/components:**
  - `domain/` enums + criteria
  - `application/` use-case request model
- **Implementation notes:** Names must match API encoding decision.
- **Acceptance criteria:**
  - Criteria model supports optional filters.
  - Unit tests demonstrate semantics.
- **Verification method:** Unit test execution.
- **Estimated effort:** 4–8 hours (depends on existing structure).

## PH1-T02 — API request parsing + validation rules

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** PH0-T02, PH1-T01
- **Relevant requirement refs:** filter selection behaviors; backward compatibility
- **Relevant design-review:** invalid cases => 400; propertyType single-select
- **Description:**
  - Parse new query params.
  - Validate:
    - invalid enum values => 400
    - too many amenities values => 400 (guardrail; limit to a small number like 10 unless existing standards differ)
    - multiple propertyType values => 400
  - Ensure missing params produce current behavior.
  - Standardize error response format per existing API conventions.
- **Expected files/components:**
  - Controller/route handler
  - Request DTO + validator
  - Error mapping/middleware
- **Acceptance criteria:**
  - Valid requests accepted.
  - Invalid propertyType multi-value rejected with 400.
  - Invalid enum rejected with 400.
- **Verification method:** Unit tests + API/contract tests.
- **Estimated effort:** 6–12 hours.

## PH2-T01 — Update search use-case to apply advanced filters

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH1-T01, PH1-T02, PH0-T03
- **Relevant requirement refs:** AC1–AC7
- **Relevant architecture:** application layer orchestrates; infra builds queries
- **Description:**
  - Extend search use-case/service to accept criteria and pass to repository.
  - Ensure composition with existing filters.
  - Ensure empty match returns empty list (client shows “No results found”).
- **Expected files/components:** `application/search*` use-case/service
- **Acceptance criteria:**
  - Combined filters behave as AND.
  - No-match returns empty list.
  - Existing search without new filters unchanged.
- **Verification method:** Integration tests with fixture data.
- **Estimated effort:** 6–12 hours.

## PH2-T02 — Infra query/repository implementation (backend-specific)

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH2-T01, PH0-T03
- **Relevant requirement refs:** AC1–AC7
- **Relevant architecture:** performance/indexing considerations
- **Description (SQL path):**
  - Map criteria to SQL predicates.
  - Amenities ALL/AND semantics (e.g., group/having count match, array containment, or join strategy depending on schema).
  - PropertyType equality predicate.
  - Ensure pagination & sorting consistent.

  **Description (search engine path):**
  - Map to bool must filters; amenities must contain all selected.
  - PropertyType term filter.

- **Expected files/components:** `infra/repository`, query builder.
- **Acceptance criteria:**
  - Correctness: amenities ALL/AND verified.
  - Performance: uses indexes/fields appropriately (as feasible).
- **Verification method:** Integration tests + query-level tests.
- **Estimated effort:**
  - SQL: 8–16 hours depending on schema.
  - Search engine: 6–12 hours depending on existing client.

## PH2-T03 — Data model/migrations (if required)

- **Phase:** 2
- **Priority:** P1
- **Dependencies:** PH0-T01, PH0-T03
- **Relevant requirement refs:** precondition: records support amenities/propertyType
- **Description:**
  - If schema does not already support required fields, add migrations:
    - propertyType column/enum
    - amenities representation (join table / array / JSON)
  - Add indexes for filter fields.
  - Add seed/test data for new attributes.
- **Expected files/components:** migrations + seed scripts.
- **Acceptance criteria:**
  - Schema supports querying required attributes.
  - Migration is reversible where applicable.
- **Verification method:** Migration apply/rollback in CI/dev; integration tests.
- **Estimated effort:** 8–20 hours (high variance).

## PH3-T01 — Contract & integration test suite

- **Phase:** 3
- **Priority:** P0
- **Dependencies:** PH1-T02, PH2-T02
- **Relevant requirement refs:** AC1–AC7
- **Description:**
  - Add integration tests for:
    - each amenity filter
    - propertyType filter
    - combined filters
    - no results
    - backward compatibility (no new params)
  - Add API contract tests for query param encoding.
- **Expected files/components:** `tests/integration/*`, `tests/contract/*`
- **Acceptance criteria:** Coverage for main scenarios and error cases.
- **Verification method:** CI test run.
- **Estimated effort:** 8–16 hours.

## PH3-T02 — Observability, security, and guardrails

- **Phase:** 3
- **Priority:** P1
- **Dependencies:** PH1-T02, PH2-T02
- **Relevant architecture:** performance/maintainability goals
- **Description:**
  - Add structured logs for validation failures and query execution timing (avoid PII).
  - Enforce pagination bounds and max filter cardinality.
  - Add rate-limit hooks if existing platform supports it (do not introduce new external services).
- **Expected files/components:** logging middleware, metrics hooks.
- **Acceptance criteria:**
  - Logs/metrics present and consistent.
  - Abuse guardrails verified by tests.
- **Verification method:** unit/integration tests; log assertion where feasible.
- **Estimated effort:** 4–10 hours.

## PH3-T03 — Documentation updates

- **Phase:** 3
- **Priority:** P1
- **Dependencies:** PH0-T02, PH1-T02
- **Description:**
  - Update API docs with new filters, encoding examples, and error responses.
  - Add examples for AND semantics.
- **Expected files/components:** `docs/*` (e.g., `docs/api-contract.md`) and/or README.
- **Acceptance criteria:** docs match implementation and tests.
- **Verification method:** PR review.
- **Estimated effort:** 2–4 hours.

---

# Dependency Matrix

| Task | Depends on |
| --- | --- |
| PH0-T02 | PH0-T01 |
| PH0-T03 | PH0-T01 |
| PH1-T01 | PH0-T02 |
| PH1-T02 | PH0-T02, PH1-T01 |
| PH2-T01 | PH1-T01, PH1-T02, PH0-T03 |
| PH2-T02 | PH2-T01, PH0-T03 |
| PH2-T03 | PH0-T01, PH0-T03 |
| PH3-T01 | PH1-T02, PH2-T02 |
| PH3-T02 | PH1-T02, PH2-T02 |
| PH3-T03 | PH0-T02, PH1-T02 |

---

# Requirements Traceability Matrix

> The requirements doc does not define IDs; traceability is by requirement statement/acceptance-criteria text.

| Requirement / Acceptance criteria (from `requirement.md`) | Tasks |
| --- | --- |
| Add amenity filters: Free Wi‑Fi, Breakfast included | PH1-T01, PH1-T02, PH2-T01, PH2-T02, PH3-T01 |
| Add property type filters: Hotel, Villa | PH1-T01, PH1-T02, PH2-T01, PH2-T02, PH3-T01 |
| Both Amenities + PropertyType match all selected criteria | PH1-T01, PH2-T01, PH2-T02, PH3-T01 |
| No matches => “No results found” (empty list from API) | PH2-T01, PH3-T01 |
| Backward compatibility with existing search | PH1-T02, PH2-T01, PH3-T01 |
| propertyType multi-select invalid => 400 | PH1-T02, PH3-T01 |

---

# Testing Strategy

1. **Unit tests**
   - Criteria semantics (amenities ALL/AND; AND across categories).
   - Validation: invalid enums, multiple propertyType values.
2. **Integration tests**
   - End-to-end search with seeded data to validate filtering.
   - Pagination bounds.
3. **Contract tests**
   - Query param encoding agreed in PH0-T02.
4. **Non-functional checks**
   - Basic performance sanity for filtered queries (timing logs/metrics).
   - Concurrency: ensure repository/query code is stateless and safe under parallel requests.

---

# Definition of Done

- `impl-plan.md` approved.
- PH0 decisions (API contract + backend) approved.
- Implementation merged with:
  - new filters working and validated
  - backward compatibility proven by tests
  - error cases return 400 as defined
  - tests passing in CI
  - documentation updated

---

# Risks and Mitigations

- **Risk:** Codebase not present in this repo branch.
  - **Mitigation:** PH0-T01 to identify merge source; do not start PH1–PH3 coding until resolved.
- **Risk:** Backend choice unclear causes rework.
  - **Mitigation:** PH0-T03 before infra tasks.
- **Risk:** Amenities ALL/AND semantics can be expensive in SQL depending on schema.
  - **Mitigation:** Add appropriate indexes; choose containment strategy suited to schema.

---

# Open Questions and Blockers

1. Where is the actual accommodation search service code for this project (this repo currently has docs only)?
2. What is the current search backend (SQL vs search engine vs vendor API)?
3. What is the existing endpoint path and pagination convention?
4. Do accommodations already store amenities/propertyType and are they populated?
5. Any auth/rate limit requirements for the search endpoint?

---

# Implementation Sequence

1. **PH0-T01** (discover code + current backend)
2. **PH0-T02** (finalize API contract)
3. **PH0-T03** (confirm backend approach)
4. **PH1-T01 → PH1-T02** (domain model + validation)
5. **PH2-T01 → PH2-T02** (use-case + infra query)
6. **PH2-T03** (migrations if needed)
7. **PH3-T01 → PH3-T03** (tests, observability, docs)

---

# Final Planning Summary

This plan introduces advanced accommodation search filters for Amenities (Free Wi‑Fi, Breakfast included) and Property Type (Hotel, Villa) while preserving backward compatibility. The critical path is closing PH0 decisions (API contract + backend) and then implementing validation/semantics (amenities ALL/AND, propertyType single-select) followed by backend-specific query changes and a comprehensive integration/contract test suite.
