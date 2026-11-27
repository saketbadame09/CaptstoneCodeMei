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
| Add amenity filters: Free Wi‑Fi, Breakfast included | Domain enum for amenities; infra query mapping; API contract supports many values |
| Add property type filters: Hotel, Villa | Domain enum for property type; infra query mapping |
| Selecting both Amenities and Property Type returns results matching all selected criteria | Define semantics: AND across categories; confirm ALL vs ANY within amenities |
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

**Recommendation:** Keep the API and domain model independent of the underlying engine; implement the repository against the existing backend (DB or search index).

## 6. Component diagram

```mermaid
flowchart LR
  U[User (web/mobile)] --> API[Search API / Controller]
  API --> UC[SearchUseCase]
  UC --> REPO[AccommodationSearchRepository (interface)]
  REPO --> DS[(SQL DB or Search Index)]
  API --> OBS[Logs/Metrics/Tracing]
  UC --> OBS
  REPO --> OBS
```

## 7. Component responsibilities

- **Search Controller (HTTP)**
  - Parse and validate input params/body
  - Map request to domain `SearchCriteria`
  - Map domain results to API response DTOs

- **SearchUseCase (Application)**
  - Build `SearchCriteria` from existing + advanced filters
  - Enforce filter semantics consistently (AND/OR)
  - Call repository and return paged result

- **Domain: SearchCriteria**
  - Holds: destination, date range, passengers, amenities, property types, paging/sorting
  - Defines semantics (documented; see API semantics section)

- **AccommodationSearchRepository (Interface)**
  - `search(criteria, paging): SearchResult`

- **Infrastructure repository (DB or Search index)**
  - Translate criteria to SQL/DSL
  - Apply paging/sorting
  - Return results + (optional) total

## 8. Data flow

1. User sends search request (with or without advanced filters).
2. Controller validates inputs and constructs domain `SearchCriteria`.
3. Use-case calls repository with criteria + pagination.
4. Repository executes query (SQL/ES), applying:
   - destination
   - date-range availability
   - passengers/capacity
   - amenities (new)
   - property types (new)
5. Results returned to controller → client.
6. Client shows results or “No results found” when empty.

## 9. Data model

**Fact:** Accommodation records support amenities and property types (per requirements).

Recommended domain enums:

- `AmenityCode`: `FREE_WIFI`, `BREAKFAST_INCLUDED`
- `PropertyType`: `HOTEL`, `VILLA`

Relational DB modeling options (if DB-backed):

- **Option 1 (recommended): normalized join table**
  - `accommodation_amenities(accommodation_id, amenity_code)`
  - Index: `(amenity_code, accommodation_id)` and/or `(accommodation_id, amenity_code)`

- **Option 2: array/JSONB (Postgres)**
  - Requires GIN indexes
  - Simpler schema but more vendor-specific and query coupling

Property type typically stored as enum/string column on `accommodations`.

## 10. API design

### 10.1 Endpoint

Recommended to extend existing search endpoint with optional filters (no breaking changes):

- **GET** `/api/accommodations/search`

### 10.2 Request parameters (high-level)

Existing (assumed):
- `destination`
- `checkIn`, `checkOut`
- `passengers`

New (optional):
- `amenities`: comma-separated enums, e.g. `FREE_WIFI,BREAKFAST_INCLUDED`
- `propertyTypes`: comma-separated enums, e.g. `HOTEL,VILLA`

Recommended (if not already present):
- `page`, `pageSize`
- optional `sort` (if needed/exists)

Validation:
- strict allowlist validation for amenity and property type values.

### 10.3 Semantics (OPEN)

- Across filter groups: **AND**
  - existing criteria AND amenities AND propertyTypes

- Within propertyTypes: **OR** (match any selected type)

- Within amenities: **OPEN** (needs product decision)
  - Option A: **ALL** selected amenities required (common expectation for multiple amenity checkboxes)
  - Option B: **ANY** selected amenity match (broader results)

### 10.4 Response (high-level)

Return:
- `items`: list of accommodations
- (optional) `total`: total matches for pagination

No matches:
- return `items: []` with HTTP 200; UI displays “No results found”.

## 11. Security architecture

- Strict allowlist validation for enums (amenities/property types)
- Parameterized queries / safe query builders (no string concatenation)
- Auth/authz: not specified (leave as-is from current system)
- Rate limiting (recommended) for public search endpoints

## 12. Error handling

- **400** for validation errors (unknown amenity/property type, invalid dates, etc.)
- **500** for unexpected errors; include correlation ID in logs (and optionally response)
- Timeouts/throttling: return **503/429** as appropriate if implemented

## 13. Testing strategy

- Unit tests:
  - `SearchCriteria` semantics (AND/OR rules)
  - enum validation/mapping

- Integration tests:
  - repository query generation and correctness
  - positive/negative filter combinations
  - “no results found” returns empty list

- End-to-end tests:
  - existing search still works
  - advanced filters appear, apply correctly, and no-results state is shown

## 14. Deployment architecture

Deployed within the existing application deployment unit (monolith or service).

If DB-backed:
- add indexes and migrations (if needed)

If search-index-backed:
- update index mapping and reindex as infra requires

## 15. Architectural decisions and trade-offs

- **ADR-001: Modular monolith over microservices**
  - Decision: Use layered/modular architecture
  - Rationale: limited scope; microservices add operational complexity without proportional value

- **ADR-002: Extend existing API with optional filters**
  - Decision: Add `amenities` and `propertyTypes` as optional params
  - Rationale: backward compatible; clients opt-in to new feature

- **ADR-003: Amenity multi-select semantics ALL vs ANY (OPEN)**
  - Decision: TBD
  - Impact: query generation and user expectations

## 16. Risks and limitations

- Unknown current query engine (DB vs search index) may require adaptation.
- Performance regression risk if amenity filtering adds expensive joins without indexes.
- Semantics for multi-select amenities not yet confirmed (ALL vs ANY).

## 17. Future enhancements

- Faceted filters (return counts for each amenity/property type for UI)
- Additional amenities and property types
- More filters (price range, rating, policies)
- Caching strategy for hot destinations/date ranges

---

## Open questions (tracked, not blocking this draft)

1. Current search backend: SQL DB vs Elasticsearch/OpenSearch vs other?
2. Multiple amenities selection semantics: match ALL vs ANY?
3. Existing pagination/sorting parameters and defaults?
4. Any inventory constraints (active/bookable/partner scoping) and auth requirements?
