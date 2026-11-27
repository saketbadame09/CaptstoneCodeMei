# Document Information

- **Title:** impl-plan.md — Advanced Accommodation Search Filters
- **Status:** Updated after Phase 0 decision closure (planning only)
- **Repo:** https://github.com/saketbadame09/CaptstoneCodeMei
- **Target branch:** `feature/advanced-accommodation-filters-requirements`
- **Last updated:** 2026-11-27

---

# Planning Objectives

1. Translate approved requirements into an actionable, dependency-aware implementation plan.
2. Ensure backward compatibility: existing search works unchanged when new filters are omitted.
3. Include validation, error handling, logging/observability, security/abuse guardrails, and a test strategy.
4. Maintain traceability from implementation tasks to requirements/architecture/design-review.
5. Avoid inventing business rules beyond approved documents; keep undecided items as open questions.

---

# Source Documents

Ordered by authority as applied in this repo:

1. **Approved requirements:** `requirement.md`
2. **Approved architecture:** `docs/Architecture.md` (contains open questions; treat only confirmed items as decisions)
3. **Approved design review:** `docs/design-review.md` (Approved with conditions; captures key decisions)
4. **Conversation decisions (human-approved):**
   - Implementation stack: **Java (Spring Boot)**
   - Persistence: **PostgreSQL**
   - Endpoint path: **`/accommodations/search`** (confirmed)

---

# Implementation Principles

1. **Backward compatible API**: new query params optional; omission yields existing behavior.
2. **Deterministic semantics**:
   - AND across categories (existing filters ∧ amenities ∧ propertyType).
   - **Amenities multi-select = ALL (AND)** within amenities (design-review decision).
   - **propertyType = single-select**; multi-value is **HTTP 400** (design-review decision).
3. **Edge validation**: allowlist enums; reject invalid values early with 400.
4. **Layer separation** per architecture intent: controller/transport ↔ application/use-case ↔ domain ↔ infra/persistence.
5. **Observability**: structured logs for request/validation failures (no PII); request-id correlation.
6. **Performance guardrails**: enforce pagination bounds, max filter cardinality, DB indexes.

---

# Technical Work Breakdown Structure

**Epic:** Advanced Filters in Accommodation Search (greenfield Spring Boot service)

1. Repository & solution setup (Spring Boot, build, config)
2. API contract & validation
3. Domain model (enums/criteria)
4. Persistence layer (Postgres schema, JPA/SQL queries)
5. Use-case/service wiring
6. Tests (unit/integration/contract/non-functional)
7. Documentation & CI verification

---

# Implementation Phases

- **Phase 0 (Done):** Decision closure
  - ✅ Java Spring Boot
  - ✅ PostgreSQL
  - ✅ Endpoint path `/accommodations/search`
- **Phase 1 (P0):** Repository/service bootstrap + baseline search API skeleton
- **Phase 2 (P0):** Domain model + DB schema + repository queries for advanced filters
- **Phase 3 (P1):** Test hardening, observability, documentation, CI improvements

---

# Task Details

> Note on traceability: `requirement.md` has no explicit requirement IDs; tasks reference requirement statements and the acceptance-criteria bullets.

## PH1-T01 — Initialize Spring Boot service skeleton

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** None
- **Relevant requirements:** “Accommodation search is available…” (precondition), advanced filters feature must be added.
- **Relevant architecture:** Layered separation (controller/application/domain/infra)
- **Description:**
  - Create a Spring Boot project structure.
  - Decide packaging convention reflecting layers (e.g., `com.codmei.accommodation.{api,application,domain,infra}`).
  - Add baseline dependencies: Spring Web, Validation, Spring Data JPA, PostgreSQL driver, Flyway/Liquibase, Test (JUnit 5), Testcontainers (optional but recommended).
  - Add health endpoint (optional) only if project standards require it.
- **Expected files/components:**
  - `pom.xml` (or `build.gradle`)
  - `src/main/java/...` main application class
  - `src/test/java/...` test skeleton
  - `src/main/resources/application.yml`
- **Implementation notes:**
  - Keep minimal—do not introduce external integrations.
- **Acceptance criteria:**
  - Service builds and starts locally.
- **Verification method:** `mvn test` and `mvn spring-boot:run` (or Gradle equivalents).
- **Estimated effort:** 3–6 hours.

## PH1-T02 — Define API contract for `/accommodations/search`

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** PH1-T01
- **Relevant requirements:** existing search filters + new advanced filters
- **Relevant design-review:** API contract previously underspecified
- **Description:** Define and document request/response:
  - Query params for existing filters (destination, dates, passengers) — exact names TBD (open question).
  - New params:
    - `amenities` multi-value (encoding TBD)
    - `propertyType` single value
  - Pagination params and defaults.
  - Response shape: list + pagination metadata (total TBD).
  - Error format for 400 validation failures.
- **Expected files/components:**
  - `impl-plan.md` (this section)
  - optionally `docs/api-contract.md`
- **Acceptance criteria:**
  - Contract decisions are explicit and approved.
- **Verification method:** Human review approval.
- **Estimated effort:** 1–2 hours.

## PH1-T03 — Implement request DTOs + validation + error handling

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** PH1-T02
- **Relevant requirements:** filter selection; invalid input => 400 (design review)
- **Description:**
  - Implement controller accepting `/accommodations/search`.
  - Parse query params into a request object.
  - Validate:
    - `propertyType` must be single value
    - `amenities` values must be allowlisted
    - pagination bounds
  - Implement global exception handler to return consistent 400 responses.
- **Expected files/components:**
  - `api/AccommodationSearchController.java`
  - `api/dto/AccommodationSearchRequest.java`
  - `api/error/ApiError.java`, `api/error/RestExceptionHandler.java`
- **Acceptance criteria:**
  - Invalid values return 400 with clear message.
- **Verification method:** MVC tests (`@WebMvcTest`) for validation cases.
- **Estimated effort:** 6–10 hours.

## PH2-T01 — Domain enums and filter semantics

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH1-T02
- **Relevant requirements:** amenities/property type filter options
- **Relevant design-review decisions:** amenities ALL/AND; AND across categories
- **Description:**
  - Define `Amenity` enum: WIFI, BREAKFAST (names mapped to UI labels “Free Wi‑Fi”, “Breakfast included”).
  - Define `PropertyType` enum: HOTEL, VILLA.
  - Define a `SearchCriteria` domain object.
  - Encode semantics in application layer: AND across filters; amenities require all selected.
- **Expected files/components:**
  - `domain/Amenity.java`, `domain/PropertyType.java`
  - `domain/SearchCriteria.java`
  - `application/SearchCriteriaMapper.java` (API → domain)
- **Acceptance criteria:**
  - Unit tests cover semantics.
- **Verification method:** `mvn test` (unit).
- **Estimated effort:** 4–8 hours.

## PH2-T02 — PostgreSQL schema & migrations

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH1-T01, PH2-T01
- **Relevant requirements:** accommodation records support amenities/property type
- **Description:**
  - Create baseline schema to support searching accommodations with:
    - destination (TBD representation)
    - capacity/passengers
    - propertyType
    - amenities (multi)
  - Choose amenities persistence strategy (open question): join table vs Postgres array.
  - Add indexes to support filters.
- **Expected files/components:**
  - `src/main/resources/db/migration/V1__init.sql` (Flyway) or Liquibase changelog
- **Acceptance criteria:**
  - Migrations apply cleanly on empty DB.
- **Verification method:** Start app with Postgres; verify Flyway runs.
- **Estimated effort:** 6–14 hours.

## PH2-T03 — Persistence layer (JPA entities + repository queries)

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH2-T02
- **Relevant requirements:** filtering behavior
- **Description:**
  - Define JPA entities and mappings for accommodations and amenities.
  - Implement repository query that applies:
    - destination/date/passengers (as defined)
    - propertyType equality
    - amenities ALL/AND
  - Implement pagination.
- **Expected files/components:**
  - `infra/jpa/AccommodationEntity.java`, etc.
  - `infra/jpa/AccommodationRepository.java`
  - custom query implementation if needed.
- **Acceptance criteria:**
  - Query returns correct results for each combination.
- **Verification method:** `@DataJpaTest` + Testcontainers Postgres.
- **Estimated effort:** 10–20 hours (depends on chosen schema/query approach).

## PH2-T04 — Application service/use-case wiring

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH1-T03, PH2-T03
- **Relevant requirements:** end-to-end search with advanced filters
- **Description:**
  - Implement `AccommodationSearchService` that:
    - maps API request → domain criteria
    - calls repository
    - maps entity → response DTO
  - Ensure empty match returns empty list.
- **Expected files/components:**
  - `application/AccommodationSearchService.java`
  - `api/dto/AccommodationSearchResponse.java` (and item DTO)
- **Acceptance criteria:**
  - Endpoint returns filtered results.
- **Verification method:** Integration tests (`@SpringBootTest`).
- **Estimated effort:** 6–12 hours.

## PH3-T01 — Contract tests + integration tests

- **Phase:** 3
- **Priority:** P0
- **Dependencies:** PH2-T04
- **Relevant requirements:** all scenarios + no results
- **Description:**
  - Add integration tests using Postgres (Testcontainers) for:
    - amenity filter (single)
    - amenity filter (both) => requires ALL
    - propertyType filter
    - combined filters
    - invalid propertyType multi-value => 400
    - invalid amenity => 400
    - no results => empty list
- **Expected files/components:** `src/test/java/.../AccommodationSearchIT.java`
- **Acceptance criteria:** tests cover core acceptance scenarios.
- **Verification method:** CI/local `mvn test`.
- **Estimated effort:** 8–16 hours.

## PH3-T02 — Observability & guardrails

- **Phase:** 3
- **Priority:** P1
- **Dependencies:** PH1-T03, PH2-T04
- **Description:**
  - Add request logging (method/path/status/duration) and validation failure logging.
  - Ensure logs do not contain PII.
  - Add max `pageSize` enforcement.
- **Expected files/components:** logging config + possibly servlet filter/interceptor.
- **Acceptance criteria:** logs present; guardrails tested.
- **Verification method:** integration test asserts 400 for excessive pageSize.
- **Estimated effort:** 4–8 hours.

## PH3-T03 — Documentation & CI

- **Phase:** 3
- **Priority:** P1
- **Dependencies:** PH1-T02, PH3-T01
- **Description:**
  - Update README with local run instructions (Postgres, migrations, tests).
  - Add GitHub Actions workflow (if not present) running build + tests.
- **Expected files/components:** `.github/workflows/ci.yml`, `README.md`
- **Acceptance criteria:** CI passes on PR.
- **Verification method:** GitHub Actions run.
- **Estimated effort:** 3–6 hours.

---

# Dependency Matrix

| Task | Depends on |
| --- | --- |
| PH1-T02 | PH1-T01 |
| PH1-T03 | PH1-T02 |
| PH2-T01 | PH1-T02 |
| PH2-T02 | PH1-T01, PH2-T01 |
| PH2-T03 | PH2-T02 |
| PH2-T04 | PH1-T03, PH2-T03 |
| PH3-T01 | PH2-T04 |
| PH3-T02 | PH1-T03, PH2-T04 |
| PH3-T03 | PH1-T02, PH3-T01 |

---

# Requirements Traceability Matrix

| Requirement / Acceptance criteria (from `requirement.md`) | Tasks |
| --- | --- |
| Add amenity filters (Free Wi‑Fi, Breakfast included) | PH2-T01, PH2-T03, PH2-T04, PH3-T01 |
| Add property type filters (Hotel, Villa) | PH2-T01, PH2-T03, PH2-T04, PH3-T01 |
| Selecting both amenities and property type returns matches for all criteria | PH2-T01, PH2-T03, PH3-T01 |
| No accommodations match => return empty list | PH2-T04, PH3-T01 |
| Existing filters still work (destination/dates/passengers) | PH1-T02, PH2-T03, PH3-T01 |
| Invalid multi-value propertyType => 400 | PH1-T03, PH3-T01 |

---

# Testing Strategy

- **Unit:** domain semantics, enum mapping, validators.
- **Web/MVC:** 400 error scenarios and request parsing.
- **JPA:** repository query correctness using Postgres.
- **Integration:** `/accommodations/search` end-to-end with seeded data.
- **Non-functional:** pagination limit and basic concurrency (parallel test execution) to ensure statelessness.

---

# Definition of Done

- Service skeleton exists and runs.
- `/accommodations/search` implemented with advanced filters.
- Validation and 400 error handling implemented per design-review decisions.
- Postgres migrations defined and reproducible.
- Tests pass locally and in CI.
- Docs updated.

---

# Risks and Mitigations

- **Underspecified existing filters contract (destination/dates/passengers):**
  - Mitigation: finalize names/types in PH1-T02; keep changes minimal.
- **Amenities ALL/AND query complexity:**
  - Mitigation: choose schema to support containment efficiently; index accordingly.
- **Greenfield scope exceeds original assumption of “add filters to existing system”:**
  - Mitigation: align expectations; ensure base search functionality is in scope.

---

# Open Questions and Blockers

1. Confirm **amenities query param encoding**: repeated vs CSV.
2. Confirm **pagination contract**: `page/pageSize` vs `offset/limit`, defaults, max.
3. Confirm whether response must include **`total`** count.
4. Specify exact contract for existing filters: destination format, date representation, passengers field.
5. Any authentication/authorization requirements? (none stated in approved docs).

---

# Implementation Sequence

1. PH1-T01 (bootstrap Spring Boot)
2. PH1-T02 (finalize API contract)
3. PH1-T03 (controller + validation + error handling)
4. PH2-T01 (domain semantics)
5. PH2-T02 (migrations)
6. PH2-T03 (repository queries)
7. PH2-T04 (use-case/service wiring)
8. PH3-T01 (integration/contract tests)
9. PH3-T02 (observability/guardrails)
10. PH3-T03 (docs/CI)

---

# Final Planning Summary

We will implement the Advanced Accommodation Search Filters as a **new Spring Boot + PostgreSQL** service exposing `GET /accommodations/search`. The plan prioritizes contract finalization and validation semantics (amenities ALL/AND; propertyType single-select) before implementing Postgres schema/query logic and comprehensive integration tests. Remaining open questions are limited to API encoding/pagination/total and the exact shape of the legacy filters.
