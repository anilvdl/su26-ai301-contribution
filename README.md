# AI-301 Open Source Capstone — Contribution README

**Student:** Anil Kumar Veldurthi <br/>
**Program:** AI301 | AI Open Source Capstone (Summer 2026, Section 1B)<br/>
**Member ID:** 153374<br/>
**Project contributed to:** [carlos-emr/carlos](https://github.com/carlos-emr/carlos) (Carlos EMR)

This README documents two open-source contributions to the Carlos EMR project:

1. **[PR #2938](https://github.com/carlos-emr/carlos/pull/2938)** - Convert `ConsultationRequestDaoImpl.getConsults` to the type-safe JPA Criteria API (issue #1748). PR open, all checks green. `yingbull` approved at first review; that approval was dismissed when the pre-merge merge-conflict resolution (a DCO-signed merge commit onto latest `develop`) was pushed, so the PR is awaiting `yingbull`'s re-review plus the second maintainer approval required by branch protection.
2. **[PR #2977](https://github.com/carlos-emr/carlos/pull/2977)** — Fix HTTP 500 on bulk tickler complete/delete caused by unsafe Jackson `ArrayNode` parsing (issue #2958). Merged to `develop` on 2026-06-23.

**Status as of Week 7 (2026-07-19):** #2977 is merged to `develop` (merge commit `40986ff`; 29 CI checks passed). #2938 remains open with all checks green; nothing has moved since last week. `yingbull`'s first-review approval was dismissed when the pre-merge merge-conflict resolution (a DCO-signed merge commit onto latest `develop`) was pushed, so the PR is awaiting `yingbull`'s re-review and the second approval required by branch protection.

---

# Contribution #1: Investigate: Hibernate 7 JPA 3 Criteria API (Type-Safe Queries)

**Contribution Number:** 1 </br>
**Student:** Anil Kumar Veldurthi </br>
**Issue:** https://github.com/carlos-emr/carlos/issues/1748 </br>
**Pull Request:** https://github.com/carlos-emr/carlos/pull/2938 </br>
**Status:** Phase IV Complete - PR open, all checks green. `yingbull` approved at first review with no change requests; that approval was dismissed when the pre-merge merge-conflict resolution (a DCO-signed merge commit onto latest `develop`) was pushed, so the PR is awaiting `yingbull`'s re-review and the second maintainer approval required by the repo's two-approval branch-protection rule before merge.

---

## Why I Chose This Issue

The Carlos EMR codebase builds its database queries as string-based HQL (for example, `"from Demographic where locked <> '1'"`), and its complex healthcare search screens push developers toward dynamically concatenating query fragments - a pattern that is fragile and creates a real HQL/SQL-injection risk whenever a user-influenced value reaches the query string without perfect parameterization. This issue asks for a focused investigation and refactor: take one complex search DAO method that builds its query through dynamic string concatenation and convert it to the type-safe Jakarta Persistence (JPA 3) `CriteriaBuilder` API, which structurally prevents injection because the query is defined by typed objects and methods rather than raw strings.

I chose it because it sits directly in my area of strength - I work extensively with Java, Spring, and the JPA/Hibernate persistence stack - while still teaching me something new: applying the type-safe Criteria API to harden a real-world, legacy healthcare codebase. It also has a clear security dimension (eliminating injection surface), and its scope is bounded to a single method, which makes it a realistic, completable contribution. My learning goals are to understand how a mature EMR structures its DAO layer, to practice converting imperative string-built queries into type-safe Criteria queries, and to be able to articulate the readability and safety tradeoffs clearly.

---

## Understanding the Issue

### Problem Description
The DAO (Data Access Object) layer in Carlos constructs Hibernate queries as plain strings. For simple, fully-parameterized queries this is acceptable, but the application's complex search screens lead developers to assemble query fragments dynamically through string concatenation. Dynamic string concatenation is error-prone and opens the door to HQL/SQL injection if any user-influenced value reaches the query string without correct parameterization. The issue asks to investigate and demonstrate the Jakarta Persistence `CriteriaBuilder` API as a type-safe alternative for these dynamic queries.

### Expected Behavior
Complex dynamic queries should be built programmatically through the JPA 3 `CriteriaBuilder` API, where the query structure is expressed as typed objects and method calls. This makes the query type-safe at compile time and structurally prevents injection, because no raw string is being assembled or executed.

### Current Behavior
Queries - including dynamically-built search queries - are constructed by concatenating HQL string fragments. This relies on every developer parameterizing perfectly, every time, and the dynamic-concatenation pattern in the search screens is exactly where that discipline is most likely to slip.

### Affected Components
The persistence/DAO layer of Carlos (Java, Hibernate/JPA). Specifically, **one** complex search DAO method that currently builds its query via dynamic string concatenation - the exact method to be identified during Phase II investigation. (Carlos is a Java 21 / Spring application; the relevant code lives in the DAO classes that issue HQL.)

---

## Phase II: Reproduce & Plan

### Environment Setup
Cloned the fork and built locally - **`mvn clean install` completes with BUILD SUCCESS** and the full test suite passes. A small number of pre-existing tests initially failed due solely to a macOS `/var` -> `/private/var` symlink interacting with the production path canonicalization (`PathValidationUtils.getCanonicalFile()`, which is intentional security behavior). Those failures are environment-specific, unrelated to this issue, and were accommodated locally only - they are **not** part of this contribution branch or PR.

The build confirms the stack the refactor must target (verified against the working tree):

- Java 21 (`pom.xml:1448-1449`)
- Spring 7.0.7 (`pom.xml:100`), Spring Security 7.0.5 (`pom.xml:101`)
- Hibernate ORM 7.2.7.Final (`pom.xml:248`)
- Jakarta Persistence 3.2.0 (`pom.xml:252-254`); namespace `jakarta.persistence.*` (`javax.persistence.*` = 0 files)
- No static metamodel generator (`hibernate-jpamodelgen`) configured and no generated `*_.java` -> the refactor uses **string attribute names** (`root.get("sendTo")`), not a generated metamodel
- Existing in-repo JPA Criteria precedent to mirror: `ProgramDaoImpl.java:457-525`
This confirms the issue's "Hibernate 7 / JPA 3" premise (the original instruction-packet stack of Spring 5.3.39 / Hibernate 5.6.15 is stale - the codebase has already migrated). The usable API today is `jakarta.persistence.criteria.*` via `EntityManager.getCriteriaBuilder()`; no Hibernate upgrade is pending.

### Affected Method (identified)
`ConsultationRequestDaoImpl.getConsults(String team, boolean showCompleted, Date startDate, Date endDate, String orderby, String desc, String searchDate, Integer offset, Integer limit)`
at `src/main/java/io/github/carlos_emr/carlos/commn/dao/ConsultationRequestDaoImpl.java:76`.

Selected from a full survey of dynamically-assembled HQL across all `*Dao*.java` files: it is the **only** method that concatenates a request-derived value (`team`) directly into HQL. Every other complex dynamic search (e.g. `DemographicDaoImpl.findByAttributes`, `TicklerDaoImpl.getTicklerQueryString`) is already parameter-bound, so this is the single method where "type-safe Criteria prevents injection" is concrete rather than cosmetic.

### Steps to Reproduce
1. In the fork, open `src/main/java/io/github/carlos_emr/carlos/commn/dao/ConsultationRequestDaoImpl.java` and locate `getConsults(...)` at line 76.
2. Observe the conditional HQL assembly: when `team` is non-empty, the value is concatenated **raw** into the query - `and cr.sendTo = '<team>'` - at line 91, with no parameter binding. *(Verified by inspection, at branch time - see Phase III update below.)*
3. Confirm the contrast that isolates the vector: the date inputs are rendered from `java.util.Date` (cannot carry a quote), and `orderby`/`desc`/`searchDate` are compared to fixed tokens that map to hardcoded column names - so ORDER BY is already effectively whitelisted and is **not** injectable. The `team` concatenation at `:91` is the sole string-injection vector in this method.

### Reproduction Evidence
- **Working branch:** https://github.com/anilvdl/carlos/tree/fix-issue-1748
- **Local build:** `mvn clean install` -> BUILD SUCCESS, full test suite passing (verified locally, 2026-06-13)
- **Raw-concatenation site (verified by inspection, at branch time):** `ConsultationRequestDaoImpl.java:91`
- **Request -> DAO data flow (INFERRED - not fully traced):** `sendTo` is a Struts action property (`EctViewConsultationRequests2Action.java:58`, setter `:114`) -> request attribute `teamVar` (`:102`) -> read in JSP as `team` (`ViewConsultationRequests.jsp:169`) -> passed to `getConsults(team, ...)` (`EctViewConsultationRequestsUtil.java:122`) -> concatenated at `:91`. The end-to-end attacker-controllability of `team` assumes standard Struts param binding with no server-side allow-list; the exact Struts mapping XML was **not** traced, so this was treated as an inferred (not confirmed) exploit.
- **Update (Phase III - discovered during the pre-merge rebase):** Upstream PR **#2898** ("replaced string-formatted date literals with named parameters") landed on `develop` *after* this branch was created, and had already moved `team` (and the dates) to **named parameters** on the string query. So the raw concatenation documented above reflects the state at branch time; on current `develop` the injection vector was already mitigated. The refactor therefore **completes the migration to the type-safe Criteria form** (the issue's actual ask) rather than introducing the parameterization. The injection-probe test is retained as a binding regression guard.

---

## Solution Approach

### Analysis
The root concern is a query-construction **pattern** - dynamic HQL assembled by string concatenation. Expressing the query through the type-safe `CriteriaBuilder` API removes the string assembly entirely (compile-time checked, structurally injection-proof) - which is what issue #1748 asks for. (Upstream #2898 separately parameterized the string query; this PR completes the move to the Criteria form.)

### Proposed Solution
Refactor `getConsults` to build its query with `CriteriaBuilder` / `CriteriaQuery`, producing an equivalent result set. Preserve the public method signature and current behavior. Mirror the existing in-repo precedent at `ProgramDaoImpl.java:457-525`.

### Implementation Plan (UMPIRE)

**Understand:** Replace the dynamic-HQL `getConsults` with a type-safe JPA 3 `CriteriaBuilder` implementation that returns equivalent results.

**Match:** Follow the in-repo precedent `ProgramDaoImpl.java:457-525` (`getCriteriaBuilder()` -> `CriteriaQuery<T>` -> `Root<T>` -> `List<Predicate>` -> `select/where/orderBy`). Use `jakarta.persistence.criteria.*` and **string attribute names** (no `jpamodelgen` build change).

**Plan:**
1. Base query: `CriteriaBuilder` -> `CriteriaQuery<ConsultationRequest>` -> `Root<ConsultationRequest>`.
2. Predicates (behavior-preserving): `if (!showCompleted)` -> `cb.notEqual(root.get("status"), "4")` (preserves NULL exclusion); `team` non-empty -> `cb.equal(root.get("sendTo"), team)` (bound); date bounds -> `cb.greaterThanOrEqualTo` / `cb.lessThanOrEqualTo` binding a real `Date`.
3. Joins: `professionalSpecialist` (mapped `@ManyToOne`) -> native `root.join("professionalSpecialist", JoinType.LEFT)`. The three unmapped sort joins (`ConsultationServices`, `Demographic`, `Provider`) require Hibernate's entity-join extension (standard JPA Criteria can't join unmapped entities); the dead `ConsultationRequestExt` join is dropped. *(See Implementation Notes for the decision - "Option A".)*
4. ORDER BY: formalize the existing hand-rolled token whitelist (`"1"`-`"9"`) into a map of column paths; default/unknown token -> `referralDate desc`; preserve the secondary sort.
5. Pagination: preserve `offset` (null -> 0), `limit` (null -> 100), and the `MAX_LIST_RETURN_SIZE = 5000` cap.

**Implement:** commits [`1237f0b`](https://github.com/anilvdl/carlos/commit/1237f0b4a4919591021641876d143c6461670b4d) (refactor) + [`5378570`](https://github.com/anilvdl/carlos/commit/5378570ee46785ba0e57d65a46e83a6ccd5f4502) (tests) + [`4c7d100`](https://github.com/anilvdl/carlos/commit/4c7d10017ed4e0c98725b9b1d132af6bd3392b52) (review follow-ups) on `fix-issue-1748`.

**Review:** Follow Carlos contribution guidelines - commits DCO-signed (`git commit -s`), no AI attribution, target `develop`, GPL v2. Public signature and behavior unchanged (except the deliberate, flagged dead-join removal).

**Evaluate:** Equivalence-test old vs refactored against the same H2 dataset (see Testing Strategy); the `team` binding test returns only literal matches.

---

## Testing Strategy

### Integration Tests (run against H2, autowiring the real `consultationRequestDao` bean)
All passing - `Tests run: 17, Failures: 0, Errors: 0`:
- [x] Equivalence: no filters → all rows, `referralDate desc`, capped at 100
- [x] `showCompleted=false` with `status` ∈ {`'4'`,`'1'`,`NULL`} → NULL rows excluded
- [x] `team="cardiology"` → equality match
- [x] **Parameter binding:** `team="x' OR '1'='1"` → only literal matches (confirms the value is bound, not concatenated; verified via generated SQL: `where cr1_0.sendTo = ?`)
- [x] Date bounds (start/end × `searchDate` "1"/"0", incl. NULL dates) → inclusive bounds + NULL exclusion preserved
- [x] ORDER BY tokens `"1"`–`"9"` × direction + unknown token, incl. token 6 (`specialist.lastName`, native association join) and the secondary `service.serviceDesc` tie-break
- [x] Pagination edges: offset past end → empty; `limit > 5000` → clamped; `limit=0` → empty
- [x] **De-duplication:** a consult with multiple `ConsultationRequestExt` rows now returns once (proves the dead-join removal)

### Regression
- [x] Existing `ConsultationRequestDaoIntegrationTest` + `ConsultRequestDaoIntegrationTest` + `ConsultationServicesDaoIntegrationTest` → `Tests run: 16, Failures: 0, Errors: 0`

---

## Implementation Notes

### Progress
Implemented the refactor via a Claude + Claude Code workflow (I drafted structured instructions; CC executed locally; I validated each result before accepting it). The work split into: the predicate/pagination refactor (straightforward), a genuine design blocker on the joins, a production-wiring investigation, and a pre-merge rebase that surfaced upstream #2898.

**The join blocker and decision (Option A).** Three of the four joins in `getConsults` (`ConsultationServices`, `Demographic`, `Provider`) target *unmapped* FK scalars and are used only in `ORDER BY`. Standard JPA Criteria can only join mapped associations, so it physically cannot express these joins - and the naive workaround (extra roots + WHERE correlation) silently converts `LEFT JOIN` to `INNER`, dropping rows. CC correctly stopped and surfaced this rather than guessing. I chose **Option A**: keep `professionalSpecialist` as a native association join, use Hibernate's entity-join extension (`JpaRoot.join(Entity.class, JoinType.LEFT).on(...)`) for the three unmapped sort joins to preserve `LEFT` semantics, and **drop the dead `ConsultationRequestExt` join** (never read, only multiplied rows - an unintended duplication). Both non-standard choices are flagged in the PR for maintainer review rather than decided silently.

**Production-wiring verification.** `ConsultationRequestDaoImpl` has no `@Repository`, which raised the question of whether the method is even reachable. Tracing it: the registered `@Repository("consultationRequestDao")` bean `ConsultationRequestMergedDemographicDaoImpl` extends `ConsultationRequestDaoImpl` and does **not** override the 9-arg `getConsults` - so it inherits the refactored method, `@PersistenceContext` is injected, and the method is reached via `EctViewConsultationRequestsUtil`. Confirmed reachable production code, not dead. The test was updated to `@Autowired` the genuine bean rather than hand-construct the DAO, so it exercises the real resolution path.

**Rebase and #2898.** Before opening the PR, I rebased onto the latest `develop`, which conflicted in `getConsults` - upstream #2898 had parameterized the same lines after I branched. Resolving it confirmed the injection vector was already mitigated on `develop`, so I reframed the PR (and this README) around the issue's actual ask - adopting the type-safe Criteria API - rather than claiming a novel injection fix, and acknowledged #2898 in the PR description.

**Automated review round (PR #2938).** The PR drew several automated reviewers - cubic and CodeRabbit (no actionable findings), Sourcery (rate-limited), gemini-code-assist (three inline comments), plus a CodeFactor checkstyle run. I evaluated each on its merits rather than auto-applying:
- **Null `team` guard** (gemini - valid): `if (!team.isEmpty())` → `if (team != null && !team.isEmpty())`, so a null `team` now skips the filter, consistent with the empty-string case. Applied in `4c7d100`.
- **BDD test-method naming** (gemini + CI `bdd-test-naming.yml` - valid): collapsed seven method names with multiple underscores to the single-underscore `should<Action>_<condition>` convention. Applied in `4c7d100`.
- **Checkstyle one-statement-per-line** (CodeFactor, 46 findings - valid): split the packed fixture lines. Applied in `4c7d100`.
- **"Move the test to `src/test-modern/`"** (gemini - *incorrect*): declined. Verified against the live tree that there is no `src/test-modern/` directory; all 337 integration tests (including the sibling DAO tests in this same package) live under `src/test/`, so the file was left co-located with its peers and the evidence noted on the thread. A reminder to validate bot suggestions rather than apply them blindly.
- **Docstring coverage 25.81% < 80%** (CodeRabbit soft check): added repo-aligned JavaDoc to the public `getConsults`, the test class, and the order-by helpers, but deliberately did not document individual test methods - the repo's own convention relies on `@DisplayName` there, so chasing the 80% number would contradict it.

### Code Changes
- **Files modified:**
  - `src/main/java/io/github/carlos_emr/carlos/commn/dao/ConsultationRequestDaoImpl.java`
  - `src/test/.../ConsultationRequestGetConsultsCriteriaIntegrationTest.java` (new)
- **Commits** (branch [`fix-issue-1748`](https://github.com/anilvdl/carlos/tree/fix-issue-1748), rebased onto latest `develop`):
  - [`1237f0b`](https://github.com/anilvdl/carlos/commit/1237f0b4a4919591021641876d143c6461670b4d) - `refactor: convert ConsultationRequestDaoImpl.getConsults to JPA Criteria`
  - [`5378570`](https://github.com/anilvdl/carlos/commit/5378570ee46785ba0e57d65a46e83a6ccd5f4502) - `test: add equivalence, injection, and ORDER BY tests for getConsults Criteria refactor`
  - [`4c7d100`](https://github.com/anilvdl/carlos/commit/4c7d10017ed4e0c98725b9b1d132af6bd3392b52) - `test: align BDD naming + checkstyle, add JavaDoc; guard null team` (review follow-ups)
- **Approach decisions:** build the query with the type-safe Criteria API (the issue's ask); bind `team` and dates through the API (completing the migration; #2898 had already parameterized them on `develop`); preserve `status`/NULL/ordering/pagination behavior exactly; Option A for the joins (Hibernate entity-join extension + drop dead `ext`); both non-standard choices flagged in the PR.

### Challenges Faced
- Standard JPA Criteria can't join unmapped entities → required Hibernate's entity-join extension (no precedent in the codebase) to preserve `LEFT JOIN` semantics.
- The DAO impl isn't directly a Spring bean → had to trace the `@Repository` subclass to confirm reachability and make the test representative.
- A pre-merge rebase conflict revealed upstream #2898 had already parameterized the method → reframed the PR around the type-safety/Criteria-adoption goal and resolved the conflict by re-layering the refactor on develop's changes.

---

## Pull Request

**PR Link:** https://github.com/carlos-emr/carlos/pull/2938 </br>
**PR Title:** refactor: convert ConsultationRequestDaoImpl.getConsults to type-safe JPA Criteria (#1748) </br>
**PR Description:** Migrates `getConsults` to the type-safe JPA `CriteriaBuilder` API per #1748, building on #2898; flags two decisions for review (Hibernate entity-join extension for the unmapped sort joins; dropped dead `ext` join / de-duplication). Includes 17 integration tests + regression. </br>
**Automated review:** addressed in `4c7d100` (null-`team` guard, BDD test naming, checkstyle one-statement-per-line, JavaDoc); one suggestion (move to `src/test-modern/`) declined with evidence - see Implementation Notes. </br>
**Maintainer Feedback:** `yingbull` approved at first review, with no change requests or open concerns. That approval was dismissed when the pre-merge merge-conflict resolution (a DCO-signed merge commit onto latest `develop`) was pushed, and re-review was requested. All CI checks pass and all review threads are resolved; the PR is awaiting `yingbull`'s re-review and, under the repository's two-approval branch protection, a second maintainer approval before merge. </br>
**Status:** Phase IV Complete - PR open, automated review addressed; awaiting `yingbull`'s re-review and a second approval to merge

---
---

# Contribution #2: Fix HTTP 500 on Bulk Tickler Complete/Delete (Jackson ArrayNode Parsing)

**Contribution Number:** 2 </br>
**Student:** Anil Kumar Veldurthi </br>
**Issue:** https://github.com/carlos-emr/carlos/issues/2958 </br>
**Pull Request:** https://github.com/carlos-emr/carlos/pull/2977 </br>
**Status:** Complete - PR merged to `develop` on 2026-06-23 (merge commit `40986ff`) by maintainer `yingbull`, who publicly endorsed it on the PR ("Very helpful contribution - much welcome. Thank you."). Automated review addressed; all 29 CI checks passed; merged.

---

## Why I Chose This Issue

Issue #2958 reports that the bulk tickler operations in Carlos - "complete" and "delete" across a selected set of ticklers - returned an HTTP 500 for every valid request, making those two REST endpoints completely unusable. Ticklers are the EMR's task/reminder mechanism, so a clinician trying to clear or close out a batch of follow-up items would hit a server error every time. The defect was also reported a second time as #2823, which confirms it was a real, user-visible breakage rather than an edge case.

I chose it because it sits squarely in my strengths - Java, Spring, and REST/JSON request handling - while being tightly bounded: a single broken request-parsing path with a clear, reproducible failure and an obvious correctness contract (a valid bulk request should succeed, not 500). It is the kind of issue where the fix is small but the verification has to be careful, which matched how I want to contribute: a real reliability fix to a widely-used healthcare codebase, with regression tests that prove the endpoints actually work afterward. My learning goals were to understand how Carlos structures its REST/web-service layer, to trace a 500 down to its root cause in Jackson `JsonNode` handling, and to harden the input parsing so the endpoints fail loudly and clearly on bad input instead of silently misbehaving.

---

## Understanding the Issue

### Problem Description
The bulk tickler endpoints `POST /tickler/complete` and `POST /tickler/delete` accept a JSON array of tickler IDs. The handler iterated the incoming Jackson `ArrayNode` and cast each element directly to `Integer`. Iterating an `ArrayNode` yields `JsonNode` elements, not boxed `Integer`s, so the cast threw `ClassCastException` on every element - which surfaced to the caller as an HTTP 500 for every otherwise-valid request.

### Expected Behavior
A valid bulk request (a JSON array of integer tickler IDs) should complete or delete the referenced ticklers and return a success response. Malformed input (missing array, non-array, non-integer, fractional, or out-of-range IDs) should be rejected with a clear error rather than a 500 or silent misbehavior.

### Current Behavior (at issue time)
Every call to `POST /tickler/complete` and `POST /tickler/delete` returned HTTP 500, because each `JsonNode` element could not be cast to `Integer`. The endpoints were effectively dead.

### Affected Components
The REST/web-service layer of Carlos (Java, Spring, Jackson). Specifically the bulk tickler handlers in:
- `src/main/java/io/github/carlos_emr/carlos/webserv/rest/TicklerWebService.java`

---

## Phase II: Reproduce & Plan

### Environment Setup
Built and tested locally against the fork using the same toolchain as Contribution #1 (Java 21 / Spring 7 / Hibernate 7 Carlos stack), so no additional environment notes specific to this change. The targeted suite for this contribution builds clean: `mvn test -Dtest=TicklerWebServiceUnitTest` -> `Tests run: 9, Failures: 0, Errors: 0` -> BUILD SUCCESS (verified 2026-06-28).

A full-suite `mvn clean install` on the merged `develop` worktree reports 10 failures across three unrelated classes (`AddEditDocument2ActionUnitTest`, `MiscUtilsLoggingOverrideUnitTest`, `PathValidationUtilsUnitTest`) - all file-upload / logging-config / canonical-path tests that are sensitive to the macOS `/tmp` -> `/private/tmp` symlink canonicalization and pass green on the CI Linux runners. Zero Tickler tests fail. These failures are environment-specific and unrelated to this PR.

### Affected Code (identified)
The bulk handlers for `POST /tickler/complete` and `POST /tickler/delete` in `TicklerWebService.java`. Each read the request body's `ticklers` array and converted elements to integers by casting `JsonNode` to `Integer`. The affected handlers (line numbers in the merged version):
- `completeTicklers(JsonNode json)` - `TicklerWebService.java:269` (serves `POST /tickler/complete`)
- `deleteTicklers(JsonNode json)` - `TicklerWebService.java:296` (serves `POST /tickler/delete`)
The shared ID-extractor added by this PR, `private List<Integer> extractTicklerIds(JsonNode json)`, lives at `TicklerWebService.java:334` and is called from both handlers.

### Steps to Reproduce
1. Send a valid `POST /tickler/complete` (or `/tickler/delete`) request with a JSON body containing a `ticklers` array of integer IDs.
2. Observe HTTP 500. The root cause is a `ClassCastException`: iterating the Jackson `ArrayNode` yields `JsonNode` elements, and each is cast to `Integer`, which fails.
3. Confirm the contrast that isolates the fix: the existing `updateTickler` path already reads IDs correctly via `JsonNode.asInt()` rather than casting - so the project already contained the correct idiom, and the bulk paths simply did not use it.

### Reproduction Evidence
- **Working branch:** https://github.com/anilvdl/carlos/tree/fix-issue-2958
- **Defect reports:** #2958 (this issue) and #2823 (independent duplicate report of the same 500, since closed), which together confirm a real, repeatable failure.
- **Root cause:** `ClassCastException` from casting `JsonNode` elements of an `ArrayNode` to `Integer` in the bulk complete/delete handlers.
- **Regression coverage:** the new `TicklerWebServiceUnitTest` includes happy-path tests for both endpoints plus a `ticklers`-missing / non-array / non-integer / fractional set, which would have failed against the pre-fix `Integer`-cast handlers; all 9 pass against the fixed code.

---

## Solution Approach

### Analysis
The root cause is a request-parsing bug: the handlers assumed `ArrayNode` iteration produced `Integer`s. The minimal correct fix is to read each ID with `JsonNode.asInt()`, matching the idiom already used in `updateTickler`. Beyond the immediate crash, the same parsing path also needed to reject malformed input cleanly rather than 500-ing or silently doing the wrong thing - which is what the review round (below) drove into a shared, validated extractor.

### Proposed Solution
Replace the unsafe `Integer` cast with `JsonNode.asInt()`-based reading, and centralize ID extraction in a single shared helper used by both bulk endpoints. The helper validates the payload as an integer array and rejects missing, non-array, non-integer, fractional, and out-of-range IDs with a clear error, instead of throwing or truncating. Public endpoint behavior for valid input is unchanged - the endpoints simply work.

### Implementation Plan (UMPIRE)

**Understand:** Bulk complete/delete 500 on every request because each `JsonNode` element is cast to `Integer`; the endpoints are unusable.

**Match:** Mirror the existing in-repo idiom in `updateTickler`, which already reads IDs via `JsonNode.asInt()`.

**Plan:**
1. Add a shared extractor that takes the request body, validates the `ticklers` field is an array, and parses each element to an int.
2. The extractor enforces `isIntegralNumber()` and `canConvertToInt()` per element, rejecting non-integer, fractional (e.g. `1.9`), and out-of-range values - so bad input returns a clear error rather than a `ClassCastException`, an NPE, or a silently truncated ID.
3. Route both `POST /tickler/complete` and `POST /tickler/delete` through the shared extractor.
4. Preserve existing privilege checks and success-response behavior for valid requests.

**Implement:** Changes in `TicklerWebService.java` (46 additions / 6 deletions) plus a new `TicklerWebServiceUnitTest.java` (198 lines), across 7 commits on `fix-issue-2958` (oldest to newest; the three `Merge branch 'develop'` commits are routine syncs to keep the branch current):
- [`648b746`](https://github.com/anilvdl/carlos/commit/648b746) - `fix: tickler bulk complete/delete always 500 (#2958)` (the core `JsonNode.asInt()` fix)
- [`ab0ef74`](https://github.com/anilvdl/carlos/commit/ab0ef74) - `Merge branch 'develop' into fix-issue-2958`
- [`60b0841`](https://github.com/anilvdl/carlos/commit/60b0841) - `fix: validate ticklers payload in bulk complete/delete (#2958)` (shared validated extractor)
- [`e565a0e`](https://github.com/anilvdl/carlos/commit/e565a0e) - `test: add privilege-denied test for tickler bulk delete (#2958)`
- [`48dba6e`](https://github.com/anilvdl/carlos/commit/48dba6e) - `Merge branch 'develop' into fix-issue-2958`
- [`a880396`](https://github.com/anilvdl/carlos/commit/a880396) - `fix: reject fractional tickler ids in bulk complete/delete (#2958)` (cubic P1 hardening)
- [`f764322`](https://github.com/anilvdl/carlos/commit/f764322) - `Merge branch 'develop' into fix-issue-2958`

**Review:** Carlos contribution guidelines followed - commits DCO-signed (`git commit -s`), Conventional Commits format, no PHI, target `develop`, GPL v2. No AI attribution.

**Evaluate:** Unit tests cover the complete and delete happy paths, empty-array handling, privilege-denied behavior, and rejection of invalid/fractional IDs (see Testing Strategy).

---

## Testing Strategy

### Unit Tests (new `TicklerWebServiceUnitTest`)
All passing - `Tests run: 9, Failures: 0, Errors: 0, Skipped: 0` -> BUILD SUCCESS (verified 2026-06-28). The 9 tests (in file order):
- [x] `shouldCompleteEachTickler_whenValidIdArrayProvided` - `POST /tickler/complete` happy path, valid ID array completes (no longer 500)
- [x] `shouldDeleteEachTickler_whenValidIdArrayProvided` - `POST /tickler/delete` happy path, valid ID array deletes (no longer 500)
- [x] `shouldNotCallManager_whenTicklerArrayIsEmpty` - empty `ticklers` array handled cleanly, no manager call, no 500
- [x] `shouldDenyCompletion_whenCallerLacksTicklerUpdatePrivilege` - privilege check enforced for complete
- [x] `shouldDenyDeletion_whenCallerLacksTicklerUpdatePrivilege` - privilege check enforced for delete
- [x] `shouldReturnError_whenTicklersFieldMissing` - missing `ticklers` field rejected with a clear error
- [x] `shouldReturnError_whenTicklersIsNotArray` - non-array payload rejected with a clear error
- [x] `shouldReturnError_whenTicklerIdIsNotInteger` - non-integer ID rejected with a clear error
- [x] `shouldReturnError_whenTicklerIdIsFractional` - fractional ID (e.g. `1.9`) rejected rather than silently truncated; added in response to the review round below

Run: `mvn test -Dtest=TicklerWebServiceUnitTest` -> `Tests run: 9, Failures: 0, Errors: 0`

---

## Implementation Notes

### Progress
Implemented via a Claude + Claude Code workflow (I drafted structured instructions; CC executed locally; I validated each result before accepting it). The core fix - swapping the `Integer` cast for `JsonNode.asInt()` against the `updateTickler` precedent - was small. The substantive work was hardening the parsing into a single validated extractor and proving the endpoints behave correctly across valid and malformed input.

**Automated review round (PR #2977).** The PR drew automated reviewers (Sourcery, cubic, CodeRabbit). I evaluated each finding on its merits and verified claims against the live code before acting, rather than auto-applying suggestions:
- **Fractional-ID truncation (cubic - P1, valid):** cubic flagged that a naive `asInt()` / `canConvertToInt()` approach would accept a fractional `DoubleNode` such as `1.9` and truncate it to `1`, silently operating on the wrong tickler record. I hardened the extractor to require `isIntegralNumber()` in addition to `canConvertToInt()`, so fractional and out-of-range IDs are rejected with a clear error instead of being silently coerced. Covered by a new BDD-named regression test. cubic's P1 auto-resolved after the change.
- **CodeRabbit:** returned clean (no actionable findings) after the hardening.
- **Maintainer process gate:** as a first-time fork contributor, the GitHub Actions workflows were gated behind a maintainer "approve and run" click, and maintainer `Ben-Heerema` set the PR to draft (2026-06-22) with a request that I run CodeRabbit myself before re-review. I ran CodeRabbit, addressed the findings, marked the PR ready, and requested re-review. The PR was subsequently merged by `yingbull`.

**Scope discipline.** Raw JSON request-body logging was raised as a possible addition but deliberately kept out of scope; the maintainer tracked it in follow-up issue **#2982** rather than expanding this PR. Keeping the PR focused on the 500 fix plus input validation made it a clean, reviewable change.

### Code Changes
- **Files modified:**
  - `src/main/java/io/github/carlos_emr/carlos/webserv/rest/TicklerWebService.java` (46 +/ 6 -)
  - `src/test/java/io/github/carlos_emr/carlos/webserv/rest/TicklerWebServiceUnitTest.java` (new, 198 +)
- **Branch:** [`fix-issue-2958`](https://github.com/anilvdl/carlos/tree/fix-issue-2958), 7 commits (4 substantive fix/test commits + 3 `develop` sync merges), merged to `develop` as `40986ff`.
- **Approach decisions:** read IDs via `JsonNode.asInt()` mirroring `updateTickler`; centralize parsing in one shared, validated extractor used by both bulk endpoints; reject missing/non-array/non-integer/fractional/out-of-range IDs with a clear error; preserve privilege checks and valid-request behavior unchanged.

### Challenges Faced
- The failure surfaced only as a generic HTTP 500; root-causing it required tracing the bulk handlers to the `JsonNode` -> `Integer` cast and recognizing that `ArrayNode` iteration yields `JsonNode`, not `Integer`.
- The first-pass fix was correct for the crash but still permitted silently-truncated fractional IDs; the review round (cubic P1) pushed it to a properly-validated extractor, which is the more defensible fix for a healthcare task system.
- First-time-contributor workflow gating meant the PR could not run CI until a maintainer approved the run, so the review loop required coordinating with the maintainers rather than relying on automated checks alone.

---

## Pull Request

**PR Link:** https://github.com/carlos-emr/carlos/pull/2977 </br>
**PR Title:** fix: tickler bulk complete/delete always 500 (#2958) </br>
**PR Description:** Bulk tickler complete/delete (`POST /tickler/complete`, `POST /tickler/delete`) returned HTTP 500 for every valid request because iterating a Jackson `ArrayNode` yields `JsonNode` elements, so casting each to `Integer` threw `ClassCastException`. IDs are now read with `JsonNode.asInt()` (matching `updateTickler`) and parsed through a shared extractor that validates the array and rejects missing/non-array/non-integer/fractional/out-of-range IDs. Fixes #2958; related to #2823 (duplicate report). Adds `TicklerWebServiceUnitTest` (9 tests covering both happy paths, empty array, privilege-denied, and missing/non-array/non-integer/fractional rejection). </br>
**Automated review:** Sourcery, cubic, CodeRabbit. cubic's P1 (fractional `DoubleNode` truncation) addressed by requiring `isIntegralNumber()` + `canConvertToInt()`; CodeRabbit clean. </br>
**Maintainer Feedback:** Merged to `develop` by `yingbull` on 2026-06-23 (merge commit `40986ff`, 29 CI checks passed), with a public endorsement on the PR: "Very helpful contribution - much welcome. Thank you." </br>
**Status:** Complete - merged.

---
---

## Learnings & Reflections

**Technical skills gained.** Across the two contributions I worked hands-on with the type-safe Jakarta Persistence Criteria API and Hibernate 7's entity-join extension for unmapped associations (Contribution #1), and with Jackson `JsonNode` / `ArrayNode` parsing plus defensive input validation in a Spring REST layer (Contribution #2). Both required tracing real production wiring - a `@Repository` subclass resolution chain in one case, and the request-to-handler path for two REST endpoints in the other - rather than assuming the code under test was the code that actually runs. I also got repeated practice with the open-source contribution workflow itself: DCO sign-off, Conventional Commits, targeting `develop`, and triaging automated-review findings (cubic, CodeRabbit, gemini, Sourcery) on their merits instead of applying them blindly.

**What I'd do differently next time.** Check for in-flight upstream PRs touching the same code before branching. Contribution #1's pre-merge rebase surfaced upstream #2898, which had already parameterized the method I was refactoring and forced a mid-stream reframing of the PR's narrative; catching that earlier would have saved a rebase conflict and let me frame the PR around the Criteria-adoption goal from the start. I'd also validate automated-review suggestions against the live tree sooner - one bot suggestion (move tests to a non-existent `src/test-modern/` directory) was simply wrong, and verifying first avoided a pointless change.

**Maintainer relationship.** Both PRs went to the same project and maintainers, and the merged Contribution #2 earned a public maintainer endorsement. Contributing more than once to the same codebase builds a working relationship with its maintainers, which matters more for long-term credibility than a scatter of unrelated one-off fixes.

---

**Phase IV Complete.** #2977 is merged to the upstream repository, and #2938 is open with all checks green - `yingbull` approved at first review, that approval was dismissed by the pre-merge merge-conflict resolution, and the PR is awaiting his re-review and a second approval to merge. This Contribution README documents the PR links, summaries, maintainer feedback, and status for each.
