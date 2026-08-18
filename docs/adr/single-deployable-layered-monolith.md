# ADR: Single-Deployable Layered Monolith over Distributed Services

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-001 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Orchestration · **Impact:** high · **Dependencies:** none (foundational)

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The Petclinic platform manages a veterinary clinic's owners, pets, pet types, visits, and
veterinarians/specialties. The system is realized as **exactly one deployable** — a single Spring
Boot 4.1.0 process (Java 17, Spring MVC + Thymeleaf) whose Kubernetes and Gradle name is
`petclinic` / `spring-petclinic`.

- ✅ Responsibilities are organized by **Java package** — `owner`, `vet`, `model`, `system` — inside
  one JVM, not as separately deployable units (`src/main/java/org/springframework/samples/petclinic/{owner,vet,model,system}/**`).
- ✅ There is a single Gradle root project (`settings.gradle:1`, `rootProject.name = 'spring-petclinic'`)
  and a single Maven artifact, and one Kubernetes `Deployment`/`Service` (`k8s/petclinic.yml:3,5,16,18`).
- ✅ There is **no** message broker, service-to-service client, event bus, API gateway, or second
  deployable anywhere in the tree; all coordination is in-process method calls (architecture-baseline
  §1–§2).

This is a foundational structural decision: it scopes team topology, deployment cadence, scaling
model, and coupling (all domains share one JVM and one relational schema). Nearly every other ADR in
this backlog is bounded by it. The CAKE catalog for tenant `5K4DVCTX` describes a *different*
platform — a WebPT EMR appointment-scheduling modernization program built from React micro-frontends
and Node.js bounded-context services (Scheduler Service, Waitlist Service, Calendar API, governed by
`ADR-PLATFORM-021/024/041`, `ADR-APPLICATION-057/059`, `ADR-004…013`). Per the Source-of-Truth rule,
**observed code wins**: none of that distributed topology is realized here, so it is recorded as
Future/Intended State (Not Implemented), not as the current architecture.

## Decision

**Keep the platform as a single-deployable, layered monolith** in which domain responsibilities are
separated by Java package (logical modules) inside one Spring Boot process sharing one relational
schema — rather than decomposing into independently deployable services or enforcing hard module
boundaries with a modularity framework.

Concretely: one build artifact, one JVM, one schema; presentation (Spring MVC + Thymeleaf) → domain
/ persistence (Spring Data JPA) → one relational database; all inter-module coordination is in-process
method invocation.

## Alternatives Considered

1. **Service-per-domain decomposition (owners / vets / visits as separate services).**
   *Rejected* — introduces network hops, a service call graph, distributed transactions, and
   independent deployment/operational overhead that a small, cohesive veterinary-clinic domain does
   not justify. There is no evidence of the scaling, team-autonomy, or independent-release pressure
   that would repay that cost, and the domains are tightly data-coupled (shared owners/pets/visits
   schema), which distributed services would fragment.

2. **Modular monolith with an enforced module system (e.g. Spring Modulith / JPMS boundaries).**
   *Rejected* — package-level separation already provides adequate organization at this scale; adding
   a modularity framework imposes boundary-verification tooling and ceremony without a demonstrated
   coupling problem to solve. (This remains the natural next step 🎯 if the codebase grows or module
   boundaries begin to erode — see Open Questions.)

3. **Single process with no internal separation (all classes in one flat package).**
   *Rejected* — sacrifices the readability, testability, and future-decomposition seams that the
   `owner`/`vet`/`model`/`system` package structure provides, for no benefit.

## Consequences

**Positive**
- ✅ Simple deployment and operations — one artifact, one process, one schema; a single
  `kubectl apply` deploys the whole system (`k8s/petclinic.yml`).
- ✅ No distributed-systems failure modes (no partial failures, network retries, or eventual
  consistency to reason about); every request is a single in-JVM call chain.
- Fast local development and testing; no service orchestration required to run the app.

**Negative / trade-offs**
- ✅ All domains share one JVM and one schema, so scaling is coarse-grained (scale the whole app, not
  a hot domain) and a change in one module rebuilds/redeploys the entire deployable.
- The aggregate-oriented `owner` module couples C1/C2/C3 (owners, pets, visits) in shared code and
  data ownership — see [aggregate-root-persistence-boundary](aggregate-root-persistence-boundary.md).
- Module boundaries are convention-only (package structure); nothing prevents cross-package coupling
  from accreting over time.

**Pattern relationship**
- **Instantiates** the catalog pattern *Layered Monolith with Package-Scoped Modules*
  (`docs/architecture-inventory/patterns/domain/layered-monolith-package-modules.md`, Candidate). No
  **Pattern Drift**: this ADR is the decision the pattern describes, not a divergence from it.
- **Pattern Update Proposal:** if/when this ADR is accepted, the pattern can be promoted from
  Candidate toward Approved so future PRs are drift-checked against the single-deployable constraint.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Responsibilities split by Java package, one JVM | ✅ | `src/main/java/org/springframework/samples/petclinic/{owner,vet,model,system}/**` |
| E2 | Single Gradle root project | ✅ | `settings.gradle:1` (`rootProject.name = 'spring-petclinic'`) |
| E3 | One Kubernetes Deployment/Service named `petclinic` | ✅ | `k8s/petclinic.yml:3,5,16,18` |
| E4 | No broker / service client / second deployable; in-process coordination only | ✅ | architecture-baseline §1–§2 (verified across `pom.xml`, `build.gradle`, `src/**`, `k8s/`) |
| E5 | CAKE distributed EMR ADRs govern a different platform, not realized here | ✅ | `cake_graph_query` (2026-08-18) returned `ADR-PLATFORM-021/024/041`, `ADR-APPLICATION-057/059`, `ADR-004…013` for the WebPT scheduling program; absent from this tree |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `spring-petclinic` is the sole in-scope repository and maps 1:1 to the `petclinic` deployable. | Only one clone exists in the workspace; `k8s/petclinic.yml` `metadata.name: petclinic`. | Missing repos would leave platform-level (cross-service) decisions unrecorded. | Confirm the request `repos` list and CAKE service set with the platform owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Does the CAKE WebPT scheduling/EMR modernization program represent planned future scope for this platform? | If in scope, it would force a distributed-services evolution and supersede this monolith decision. | Not implemented in code; treated as Future/Intended State (Not Implemented). | Product / architecture owner |
| Q2 | At what growth/coupling threshold should a modular-monolith framework or service extraction be revisited? | Determines when this foundational decision should be re-opened. | Revisit if module boundaries erode or independent scaling/release pressure appears. | Platform architecture owner |
