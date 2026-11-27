# Document Information

- **Title:** impl-plan.md — Advanced Accommodation Search Filters
- **Status:** Updated after API contract + stack decisions (planning only)
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
   - Endpoint path: **`GET /accommodations/search`**
   - Amenities param encoding: **repeated query param** (`amenities=WIFI&amenities=BREAKFAST`)
   - Pagination: **`page` + `pageSize`**
   - Response includes **`total`** count

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
  - ✅ Endpoint path `GET /accommodations/search`
  - ✅ amenities query param is repeated
  - ✅ pagination is `page/pageSize`
  - ✅ response includes `total`
- **Phase 1 (P0):** Repository/service bootstrap + API contract skeleton
- **Phase 2 (P0):** Domain model + DB schema + repository queries for advanced filters
- **Phase 3 (P1):** Test hardening, observability, documentation, CI improvements

---

# Task Details

> Note on traceability: `requirement.md` has no explicit requirement IDs; tasks reference requirement statements and the acceptance-criteria bullets.

## PH1-T01 — Initialize Spring Boot service skeleton

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** None
- **Relevant requirements:** precondition that accommodation search exists; advanced filters must be supported.
- **Relevant architecture:** Layered separation (controller/application/domain/infra)
- **Description:**
  - Create a Spring Boot project structure.
  - Use packaging convention reflecting layers (e.g., `com.codmei.accommodation.{api,application,domain,infra}`).
  - Add dependencies: Spring Web, Validation, Spring Data JPA, PostgreSQL driver, Flyway, Test (JUnit 5), Testcontainers Postgres.
  - Add minimal config for local Postgres.
- **Expected files/components:**
  - `pom.xml`
  - `src/main/java/...` main application class
  - `src/main/resources/application.yml`
  - `src/test/java/...` basic test
- **Implementation notes:** Keep minimal; no external services.
- **Acceptance criteria:** Service builds and starts locally.
- **Verification method:** `mvn test` and `mvn spring-boot:run`.
- **Estimated effort:** 3–6 hours.

## PH1-T02 — Finalize API contract for `/accommodations/search`

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** PH1-T01
- **Relevant requirements:** existing filters + new advanced filters
- **Relevant design-review:** API contract underspecified; invalid cases must return 400
- **Decisions (approved):**
  - `GET /accommodations/search`
  - `amenities` is repeated query param (multi-value)
  - `propertyType` is single-select (multi-value => 400)
  - Pagination uses `page` and `pageSize`
  - Response includes `total`
- **Remaining open items:** exact parameter names/types for existing filters (destination/dates/passengers).
- **Description:**
  - Document request params and examples:
    - `amenities=WIFI&amenities=BREAKFAST`
    - `propertyType=HOTEL`
    - `page=0&pageSize=20`
  - Document response envelope including `items`, `page`, `pageSize`, `total`.
  - Define max `pageSize` (recommendation: 100 unless you specify otherwise).
- **Expected files/components:** `docs/api-contract.md` (recommended) and/or update README.
- **Acceptance criteria:** Contract is explicit and approved.
- **Verification method:** Human review sign-off.
- **Estimated effort:** 1–2 hours.

## PH1-T03 — Implement controller + request DTOs + validation + error handling

- **Phase:** 1
- **Priority:** P0
- **Dependencies:** PH1-T02
- **Relevant requirements:** advanced filters; invalid input returns 400
- **Relevant design-review decisions:** amenities ALL/AND semantics; propertyType single-select; invalid => 400
- **Description:**
  - Implement Spring MVC controller for `GET /accommodations/search`.
  - Parse query params into request DTO.
  - Validate:
    - invalid enum values => 400
    - multiple `propertyType` values => 400
    - `page>=0`, `1<=pageSize<=MAX_PAGE_SIZE`
    - `amenities` length guardrail (recommendation: <= 10)
  - Implement `@ControllerAdvice` for consistent error response.
- **Expected files/components:**
  - `api/AccommodationSearchController.java`
  - `api/dto/AccommodationSearchRequest.java`
  - `api/dto/AccommodationSearchResponse.java`
  - `api/error/ApiError.java`, `api/error/RestExceptionHandler.java`
- **Acceptance criteria:**
  - Validation failures return HTTP 400 with structured payload.
- **Verification method:** `@WebMvcTest` tests for validation paths.
- **Estimated effort:** 6–10 hours.

## PH2-T01 — Domain enums and filter semantics

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH1-T02
- **Relevant requirements:** amenities/property type options
- **Relevant design-review decisions:** amenities ALL/AND; AND across categories
- **Description:**
  - Define `Amenity` enum: WIFI, BREAKFAST.
  - Define `PropertyType` enum: HOTEL, VILLA.
  - Define `SearchCriteria` domain object.
  - Ensure semantics:
    - AND across categories
    - amenities require ALL selected
- **Expected files/components:**
  - `domain/Amenity.java`, `domain/PropertyType.java`
  - `domain/SearchCriteria.java`
  - `application/SearchCriteriaMapper.java`
- **Acceptance criteria:** Unit tests cover semantics and mappings.
- **Verification method:** `mvn test`.
- **Estimated effort:** 4–8 hours.

## PH2-T02 — PostgreSQL schema & migrations

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH1-T01, PH2-T01
- **Relevant requirements:** accommodation records support amenities/property type
- **Description:**
  - Create baseline schema sufficient for searching accommodations with:
    - destination (open question: city code vs free text)
    - available date range (open question)
    - capacity/passengers (open question)
    - propertyType
    - amenities (multi)
  - Choose amenities persistence strategy:
    - **Option A (recommended):** join table `accommodation_amenities(accommodation_id, amenity)`
    - Option B: Postgres `text[]` with GIN index
  - Add indexes for propertyType and destination fields.
- **Expected files/components:** `src/main/resources/db/migration/V1__init.sql`
- **Acceptance criteria:** Migrations apply cleanly; schema supports filter queries.
- **Verification method:** Flyway runs on startup against local Postgres.
- **Estimated effort:** 6–14 hours.

## PH2-T03 — Persistence layer (JPA entities + repository queries)

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH2-T02
- **Relevant requirements:** server-side filtering; supports pagination
- **Description:**
  - Define JPA entities.
  - Implement repository query that applies:
    - propertyType equality
    - amenities ALL/AND (e.g., join + group/having or array containment)
    - pagination
  - Implement `total` count query consistent with filters.
- **Expected files/components:**
  - `infra/jpa/*.java` entities
  - `infra/jpa/AccommodationRepository.java`
  - custom repository implementation as needed
- **Acceptance criteria:** Query returns correct results; total matches.
- **Verification method:** `@DataJpaTest` + Testcontainers Postgres.
- **Estimated effort:** 10–20 hours.

## PH2-T04 — Application service/use-case wiring

- **Phase:** 2
- **Priority:** P0
- **Dependencies:** PH1-T03, PH2-T03
- **Relevant requirements:** end-to-end search with advanced filters
- **Description:**
  - Implement `AccommodationSearchService`:
    - map request → criteria
    - call repository
    - map to response DTO
  - Ensure empty match returns empty list and `total=0`.
- **Expected files/components:** `application/AccommodationSearchService.java`
- **Acceptance criteria:** Endpoint returns filtered results with correct total.
- **Verification method:** `@SpringBootTest` integration tests.
- **Estimated effort:** 6–12 hours.

## PH3-T01 — Integration + contract tests (Testcontainers)

- **Phase:** 3
- **Priority:** P0
- **Dependencies:** PH2-T04
- **Relevant requirements:** all filter scenarios and no-results
- **Description:**
  - Add integration tests using Postgres Testcontainers for:
    - amenity filter single
    - amenity filter both => requires ALL
    - propertyType filter
    - combined filters
    - invalid propertyType multi-value => 400
    - invalid amenity => 400
    - no results => `items=[]`, `total=0`
- **Expected files/components:** `src/test/java/.../AccommodationSearchIT.java`
- **Acceptance criteria:** tests cover core acceptance scenarios.
- **Verification method:** `mvn test`.
- **Estimated effort:** 8–16 hours.

## PH3-T02 — Observability & guardrails

- **Phase:** 3
- **Priority:** P1
- **Dependencies:** PH1-T03, PH2-T04
- **Description:**
  - Add request logging (method/path/status/duration) and validation failure logging.
  - Add `MAX_PAGE_SIZE` enforcement test.
- **Expected files/components:** logging config + servlet filter/interceptor.
- **Acceptance criteria:** logs present; guardrails tested.
- **Verification method:** integration tests.
- **Estimated effort:** 4–8 hours.

## PH3-T03 — Documentation & CI

- **Phase:** 3
- **Priority:** P1
- **Dependencies:** PH1-T02, PH3-T01
- **Description:**
  - Add README with local run instructions (Postgres, migrations, tests).
  - Add GitHub Actions workflow running `mvn test`.
- **Expected files/components:** `.github/workflows/ci.yml`, `README.md`
- **Acceptance criteria:** CI passes on PR.
- **Verification method:** GitHub Actions.
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
| API returns `total` count | PH1-T02, PH2-T03, PH2-T04, PH3-T01 |

---

# Testing Strategy

- **Unit:** domain semantics, enum mapping, validators.
- **Web/MVC:** 400 error scenarios and request parsing.
- **JPA:** repository query correctness using Postgres.
- **Integration:** `/accommodations/search` end-to-end with seeded data; assert `total`.
- **Non-functional:** pagination limit and basic concurrency (parallel test execution) to ensure statelessness.

---

# Definition of Done

- Service skeleton exists and runs.
- `GET /accommodations/search` implemented with advanced filters.
- Validation and 400 error handling implemented per design-review decisions.
- Postgres migrations defined and reproducible.
- `total` is returned and consistent with filtered queries.
- Tests pass locally and in CI.
- Docs updated.

---

# Risks and Mitigations

- **Underspecified existing filters contract (destination/dates/passengers):**
  - Mitigation: finalize names/types in PH1-T02; keep changes minimal.
- **Amenities ALL/AND query complexity:**
  - Mitigation: prefer join-table approach or Postgres array containment with GIN index.
- **Greenfield scope exceeds original assumption of “add filters to existing system”:**
  - Mitigation: align expectations; implement baseline search + advanced filters as the MVP.

---

# Open Questions and Blockers

1. Specify exact contract for existing filters: destination param name & format, date params format, passengers param name/type.
2. Confirm defaults and maximums: `pageSize` default and max.
3. Confirm sorting requirements (none specified in approved docs).
4. Authentication/authorization requirements? (none stated in approved docs).

---

# Implementation Sequence

1. PH1-T01 (bootstrap Spring Boot)
2. PH1-T02 (document API contract; finalize remaining open items)
3. PH1-T03 (controller + validation + error handling)
4. PH2-T01 (domain semantics)
5. PH2-T02 (migrations)
6. PH2-T03 (repository queries + total)
7. PH2-T04 (service wiring)
8. PH3-T01 (integration/contract tests)
9. PH3-T02 (observability/guardrails)
10. PH3-T03 (docs/CI)

---

# Final Planning Summary

We will implement Advanced Accommodation Search Filters as a **new Spring Boot + PostgreSQL** service exposing `GET /accommodations/search`. The API uses repeated query params for amenities, `page/pageSize` pagination, and returns `total`. The plan prioritizes contract finalization (including legacy filter param formats), validation semantics (amenities ALL/AND; propertyType single-select), then Postgres schema/query logic with correct count semantics, and comprehensive integration tests using Testcontainers.
