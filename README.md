# Contribution #1: Investigate: Hibernate 7 JPA 3 Criteria API (Type-Safe Queries)

**Contribution Number:** 1 </br>
**Student:** Anil Kumar V </br>
**Issue:** https://github.com/carlos-emr/carlos/issues/1748 </br>
**Pull Request:** https://github.com/carlos-emr/carlos/pull/2938 </br>
**Status:** Phase III Complete · Phase IV In Progress - PR open; automated review addressed (`4c7d100`); awaiting maintainer review (Actions workflows pending maintainer "approve and run")

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
**Maintainer Feedback:** _(none yet - awaiting human review; the GitHub Actions workflows are pending a maintainer "approve and run" gate that applies to first-time fork contributors)_ </br>
**Status:** Phase IV - PR open, automated review addressed, awaiting maintainer review
