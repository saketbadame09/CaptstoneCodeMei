# Architecture – Advanced Accommodation Search Filters

> Status: DRAFT of the architecture proposal. It contains open questions and assumptions. Do not treat unconfirmed items as requirements.

## 1. Overview

The product currently supports an accommodation search flow filtered by (1) destination, (2) travel dates, (3) number of passengers. This change adds advanced filters so travelers can refine search results by:

- Amenities: Free Wi‑Fi, Breakfast included
- Property Type: Hotel, Villa

Filters must compose with existing criteria and preserve backward compatibility.

## 2. Architecture goals

- Backward compatibility: existing search contract works when new filters are omitted.
- Correctness: well-defined filter semantics (AND/OR) and deterministic behavior.
- Maintainability: add new filters without large refactors.
- Testability: separate HTTP handling, business logic, and query building.
- Performance: filtering happens server-side with appropriate indexes; supports pagination.
- Extensibility: design criteria model to map to either SQL (DB) or search engine (future) without changing APIs.

## 3. Requirements traceability

| Req / Acceptance Criteria | Architectural implication |
| --- | --- |
| Add amenity filters: Free Wim‑Fi, Breakfast included | Domain enum for amenities; infra query mapping; API contract supports many values |
| Add property type filters: Hotel, Villa | Domain enum for property type; infra query mapping |
| Selecting both Amenities and Property Type returns results matching all selected criteria | Define semantics: AND across categories; amenities require ALL selected values (AND) |
| When no accommodations match, display “No results found” | API returns empty list (and optional total); client shows no-results UI |
| Existing filters still work | Add optional fields only; no breaking changes |

## 4. Architecture pattern

Recommended: **Modular monolith / layered architecture (Clean-Architecture inspired)**

Logical layers/components:

- API/HTTP layer (controllers)
- Application layer (use-cases)
- Domain layer (filter semantics, value objects, enums)
- Infrastructure layer (query builders, repositories, DB/search clients)

Rationale: scope is limited to adding filters; microservices are not justified. This separation keeps concerns clear and supports testing.

## 5. Technology stack

**Fact:** The requirements do not specify the current stack.

**Decision options (pros/cons):**


### Option A — SQL database filtering (Postgres/MySQL)
- Pros: simpler operations, strong consistency, easier to test/deploy
- Cons: can become complex as more filters are added; must index carefully (especially amenity joins/arrays)
- Rationale: best if current search is DB-backed and scope remains structured filters.

### Option B — Search engine filtering (Elasticsearch/OpenSearch)
- Pros: fast faceted filtering, easier to extend, good for relevance/ranking
- Cons: operational overhead, index sync complexity, eventual consistency
- Rationale: best if you already have a search index, or roadmap includes many filters/facets.

**Recommendation:** Keep the API and domain model independent of the underlying engine; implement the repository against the existing backend (DB"