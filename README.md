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
