# Contribution #1: Investigate: Hibernate 7 JPA 3 Criteria API (Type-Safe Queries)
 
**Contribution Number:** 1 </br>
**Student:** Anil Kumar V </br>
**Issue:** https://github.com/carlos-emr/carlos/issues/1748 </br>
**Status:** Phase I - In Progress
 
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
2. Observe the conditional HQL assembly: when `team` is non-empty, the value is concatenated **raw** into the query - `and cr.sendTo = '<team>'` - at line 91, with no parameter binding. *(Verified by inspection.)*
3. Confirm the contrast that isolates the vector: the date inputs are rendered from `java.util.Date` (cannot carry a quote), and `orderby`/`desc`/`searchDate` are compared to fixed tokens that map to hardcoded column names - so ORDER BY is already effectively whitelisted and is **not** injectable. The `team` concatenation at `:91` is the sole string-injection vector in this method.
4. *(Empirical probe - planned for Phase IV, see Testing Strategy.)* Drive `getConsults` via a focused DAO test against H2 with `team = "x' OR '1'='1"`; the current method returns the injected superset (or errors), whereas a bound parameter returns only literal matches.
### Reproduction Evidence
- **Working branch:** https://github.com/anilvdl/carlos/tree/fix-issue-1748 (currently mirrors `develop` - Phase II contains no code changes)
- **Local build:** `mvn clean install` -> BUILD SUCCESS, full test suite passing (verified locally, 2026-06-13)
- **Raw-concatenation site (verified by inspection):** `ConsultationRequestDaoImpl.java:91`
- **Request -> DAO data flow (INFERRED - not yet fully traced):** `sendTo` is a Struts action property (`EctViewConsultationRequests2Action.java:58`, setter `:114`) -> request attribute `teamVar` (`:102`) -> read in JSP as `team` (`ViewConsultationRequests.jsp:169`) -> passed to `getConsults(team, ...)` (`EctViewConsultationRequestsUtil.java:122`) -> concatenated at `:91`. The end-to-end attacker-controllability of `team` assumes standard Struts param binding with no server-side allow-list; the exact Struts mapping XML has **not** been traced, so this is treated as an inferred (not confirmed) exploit pending verification.
- **Finding:** The raw concatenation is verified; the full request-to-DAO controllability is inferred and pending confirmation (trace the Struts binding and/or run the empirical probe).
---
 
## Solution Approach
 
### Analysis
The root concern is a query-construction **pattern** - dynamic HQL assembled by string concatenation, with one unbound request-derived value (`team`). Expressing the same query through `CriteriaBuilder` with a bound `team` parameter both type-safes the query and closes the injection surface.
 
### Proposed Solution
Refactor `getConsults` to build its query with `CriteriaBuilder` / `CriteriaQuery`, producing an equivalent result set while binding `team` as a parameter. Preserve the public method signature and current behavior; the security win is binding `team`. Mirror the existing in-repo precedent at `ProgramDaoImpl.java:457-525`.
 
### Implementation Plan (UMPIRE)
 
**Understand:** Replace the dynamic-HQL `getConsults` with a type-safe JPA 3 `CriteriaBuilder` implementation that returns equivalent results and binds the `team` parameter (closing the injection surface).
 
**Match:** Follow the in-repo precedent `ProgramDaoImpl.java:457-525` (`getCriteriaBuilder()` -> `CriteriaQuery<T>` -> `Root<T>` -> `List<Predicate>` -> `select/where/orderBy`). Use `jakarta.persistence.criteria.*` and **string attribute names** (no `jpamodelgen` build change).
 
**Plan:**
1. Base query: `CriteriaBuilder` -> `CriteriaQuery<ConsultationRequest>` -> `Root<ConsultationRequest>`.
2. Predicates (behavior-preserving):
   - `if (!showCompleted)` -> `cb.notEqual(root.get("status"), "4")` (preserves NULL exclusion; keep literal as `String "4"`).
   - `if (team != null && !team.isEmpty())` -> `cb.equal(root.get("sendTo"), team)` - **bound** (the injection fix).
   - date bounds -> `cb.greaterThanOrEqualTo` / `cb.lessThanOrEqualTo` on the chosen date attribute, binding a real `Date` (also removes the DATE-vs-datetime literal fragility).
3. Joins: `professionalSpecialist` is a mapped `@ManyToOne` -> clean `root.join("professionalSpecialist", JoinType.LEFT)`. The four ad-hoc entity joins (`ConsultationServices`, `Demographic`, `Provider`, `ConsultationRequestExt`) are **not** mapped associations and are the hard part - reproduce them faithfully (behavior-preserving) and raise the cleaner conditional-join option with the maintainer.
4. ORDER BY: formalize the existing hand-rolled token whitelist (`"1"`-`"9"`) into a map of column paths; default/unknown token -> `referralDate desc`; preserve the secondary sort and current duplicate-row behavior (no `distinct` unless the maintainer approves).
5. Pagination: preserve `offset` (null -> 0), `limit` (null -> 100), and the `MAX_LIST_RETURN_SIZE = 5000` cap.
**Implement:** *(branch commits to be added in Phase III)*
 
**Review:** Follow Carlos contribution guidelines - commits DCO-signed (`git commit -s`), no AI attribution, target `develop`, GPL v2. Public signature and behavior unchanged.
 
**Evaluate:** Equivalence-test old vs refactored against the same H2 dataset (see Testing Strategy): identical result sets including order and duplicate multiplicity; the `team` injection probe returns only literal matches after binding.
 
---
 
