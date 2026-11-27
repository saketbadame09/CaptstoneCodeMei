# Independent Design Review – Advanced Accommodation Search Filters

Role: Principal Software Architect (independent, critical, evidence-based)

Branch reviewed: `feature/advanced-accommodation-filters-requirements`

Documents reviewed:
- `requirement.md` (Source doc in repo)
- `docs/Architecture.md`

## Note on document state
At the time of review, `docs/Architecture.md` was successfully fetched and reviewed. Later observations showed that the file had been corrupted/truncated in the branch history due to an incomplete update via the GitHub Contents API. This review content reflects the architecture design intent seen during the review and captures the agreed decisions made in the review conversation (amenities ALL / AND; propertyType single-select; invalid cases return 400).

---


## 1. Executive summary
The architecture draft is generally aligned with the stated requirements (add amenity and property type filters, preserve existing search filters, return empty results for no matches). However, some gaps must be resolved before implementation:

- Define multi-select amenity semantics (decision captured: **ALL (AND) within amenities** for multiple selected amenities).
- Confirm the current search backend (SQL DB vs search engine/vendor API) to avoid rework.

- Finalize the API contract (parameter encoding, pagination defaults/bounds, whether total is returned).

Review outcome: **Approved with conditions**


---


## 2. Review findings

Format used per finding:

- Finding ID
- Title
- Severity
- Architecture section
- Description
- Impact
- Recommendation
- Decision required from human


### F-001 — Requirements file name mismatch (requirements.md vs requirement.md)
- **Severity:** Medium
- **Architecture section:** N/A (repo convention /process)
- **Description:** The repo contains `requirement.md`, not `requirements.md` as specified in the review steps.
- **Impact:** Reviewers/tools may miss the correct document; consistency and automation risks.
- **Recommendation:** Standardize on one name (prefer: `requirements.md`) or update references everywhere.
 - *(Decision required)*: Confirm canonical filename and enforce references.

### F-002 — Amenity multi-select semantics were undecided (decision now captured)
- **Severity:** High
- **Architecture section:** 10.3 Semantics (OPEN), ADR reference, Risks/Open Questions
- **Description:** The architecture draft left amenity multi-select combination undecided (ALL vs ANY). This impacts query building, indexing, and user expectations.
- **Impact:** Implementation can diverge from PO expectations; hard to write deterministic tests.
 - **Recommendation:** Capture the decision in Section 10.3 and an ADR:
  - Within amenities: **AND (ALL)** within the list
  - Across categories: AND
  - Omitted/empty amenities: no filtering
- *(Decision required)*: **CLOSED** – PO chosen option A (ALL/AND).

### F-003 — Backend search engine unconfirmed (SQL database vs search index)
- **Severity:** High
- **Architecture section:** 5 Technology stack, 16 Risks, Open questions
- **Description:** Architecture outlines options but does not lock the current system of record for this increment.
- **Impact:** Performance, indexing, data model, and migration path can change materially.
- **Recommendation:** Confirm the current search implementation and lock an implementation option for this increment; keep the domain/API engine-independent.
- *(Decision required)*: Engineering lead to confirm the backend and approve Option A or B.

### F-004 – API contract underspecified (encoding, pagingl, total)
- **Severity:** Medium
- **Architecture section:** 10.2 Request parameters, 10.4 Response, 2 Goals
- **Description:** Parameter encoding (CSV vs repeated), default page/pageSize, max pageSize, default sort, whether `total` is always returned are not fully specified.
- **Impact:** Client/backend mismatch, performance regressions (e.g. expensive COUNT()), and harder testing.
- **Recommendation:** Finalize and document the contract: encoding, paging defaults/limits, sort default, total policy.
- **Decision required:** API owner to approve final contract details.

### F-005 – Data model missing constraints and migration/backfill plan
- **Severity:** Medium
- **Architecture section:** 9 Data model, 14 Deployment architecture
- **Description:** Architecture suggests normalized amenity mapping but does not explicitly call out uniqueness constraints/FKs/backfill strategy.
- **Impact:** Data quality risk (dupes, orphans); release risk due to migration uncertainty.
- **Recommendation:** Add to architecture:
  - Unique constraint on (accommodation_id, amenity_code)
  - Foreign keys to accommodations
  - Indexes aligned to filter query patterns
  - Backfill/migration strategy
- *(Decision required)*: DBA/tech lead to approve migration and constraints.

### F-006 – Security stance too generic for search abuse protection
- **Severity:** Medium
- **Architecture section:** 11 Security architecture, 12 Error handling
- **Description:** Architecture mentions parameterization and rate limiting but doesn’t specify auth model or abuse controls for search.
- **Impact:** Search endpoints are common scraping/abouse targets; reliability and cost risk.
 - **Recommendation:** Define whether endpoint is public, rate-limit policy, max pageSize, and observability (logs, alerts).
- **Decision required:** Product/Security to confirm access and quotas.


---

## 3. Requirements traceability gaps
1. Amenity multi-select semantics was missing in requirements (resolved in review conversation); must be shared as acceptance criteria.
2. Pagination/sorting is impled by architecture but not explicit in requirements; needs scope confirmation.
3. PropertyType single-select rule (and 400 on multiple values) must be reflected in requirements/AC to avoid client mismatch.


---

## 4. Security review
- Positive: enum allowlist validation and parameterized queries are the right baseline.
- Guardrails to add: rate limiting/quotas, bot protection if public, max pageSize, and logging of rejected inputs.

---

## 5. Data model review
- Recommended: normalized mapping table for accommodation → amenities.
- Missing spec: unique (accommodation_id, amenity_code), FKs, indexes for filter query patterns, backfill mechanism.


---

## 6. API review
- Must finalize: param encoding (comma-separated vs repeated), defaults for paging_params, and total policy.
- Decision captured in review conversation:
  - `propertyType` is single-select
  - multiple/CSV/repeated `propertyType` is **HTTP 400**
  - `amenities` model supports multi-value; the semantics is **AND (ALL)** within the list (accommodation must have all selected amenities).


---

## 7. Technology review
- Lifecycle: avoid locking into a search engine if current infra is DB-backed and scope is small.


---

## 8. Recommended architecture changes (prioritized)
1. *(Must)** Update architecture to remove OPEN on amenity semantics and capture ALL/AND, with examples.
2. *(Must)** Confirm the current search backend and lock the implementation option for this increment.
1. **(Should)** Finalize API param encoding and pagination/sorting contract.


---

## 9. Questions for the Product Owner
1. Confirm amenity multi-select semantics is ALL/AND in the PC acceptance criteria.
2. Should invalid amenities be 400 (most deterministic) or ignored?

---

## 10. Final recommendation
**Approved with conditions**

Conditions that must be resolved before implementation:
1. Capture amenity semantics (within amenities: ALL/AND) as part of the architecture and/or AC.
2. Confirm search backend and commit to an implementation path for this increment.