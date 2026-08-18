# ADR: In-Process Read-Through Cache over a Networked Cache

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-006 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Storage · **Impact:** medium · **Dependencies:** ADR-CANDIDATE-001 ([single-deployable-layered-monolith](single-deployable-layered-monolith.md))

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The veterinarian listing is read-mostly: vets and their specialties change rarely but the list is
displayed on every visit to the `/vets` pages. To avoid re-querying the database on every request,
the read path is fronted by a declarative cache.

- ✅ `VetRepository.findAll()` and `findAll(Pageable)` are annotated `@Cacheable("vets")` (both also
  `@Transactional(readOnly = true)`) — `src/main/java/org/springframework/samples/petclinic/vet/VetRepository.java:44-56`.
- ✅ Caching is enabled and the `vets` cache is created programmatically through the **JCache** API,
  with statistics enabled (exposed via JMX) — `src/main/java/org/springframework/samples/petclinic/system/CacheConfiguration.java:31-51`
  (`@EnableCaching`, `JCacheManagerCustomizer`, `MutableConfiguration.setStatisticsEnabled(true)`).
- ✅ The JCache provider is **Caffeine**, an in-JVM cache library — `caffeine` dependency in
  `pom.xml:82-83` and `build.gradle:45` (`runtimeOnly 'com.github.ben-manes.caffeine:caffeine'`).
- ✅ There is **no** networked cache (Redis / Memcached) dependency anywhere in `pom.xml` or
  `build.gradle`, and no cache-server manifest in `k8s/`.

Because the cache lives inside each JVM, it is per-replica: two application instances hold two
independent copies of the `vets` cache with independent eviction. The current deployment runs a
single replica (`k8s/petclinic.yml:22`, `replicas: 1`), so this is not observable today, but it
constrains horizontal scaling — see [single-deployable-layered-monolith](single-deployable-layered-monolith.md).

## Decision

**Cache the read-mostly vet listing with a declarative `@Cacheable` read-through cache backed by an
in-process Caffeine (JCache) cache local to the JVM**, rather than introducing a networked/shared
cache or leaving the read path uncached. Cache configuration is programmatic (a single `vets` cache
with JCache statistics enabled for JMX visibility); write/invalidation is not modeled because vet
data is not mutated through the application.

## Alternatives Considered

1. **A networked/distributed cache (Redis or Memcached) shared across instances.**
   *Rejected* — provides cross-replica coherency and shared eviction, but adds an external
   service to deploy, secure, monitor, and keep available, plus a network hop on every cache
   access. For a single-replica deployment of a small, rarely-changing dataset, that operational
   overhead is not justified. (This is the natural upgrade 🎯 if the app is scaled to many replicas
   and cache coherency becomes material — see Q1.)

2. **No cache at all (query the database on every request).**
   *Rejected* — the vet list is read on effectively every page and changes rarely, so an
   in-process cache removes repeated identical queries at near-zero cost and complexity; omitting it
   trades that easy win for nothing.

3. **Manual/ad-hoc caching in the service layer (hand-rolled maps, no abstraction).**
   *Rejected* — Spring's `@Cacheable` + JCache gives a standard, declarative, provider-swappable
   abstraction (Caffeine today, a distributed provider later) with built-in statistics; a bespoke
   map forfeits all of that and invites concurrency bugs.

## Consequences

**Positive**
- ✅ Zero operational overhead — no cache server to run; the cache is a library inside the app
  (`caffeine`, `pom.xml:82-83` / `build.gradle:45`).
- ✅ Declarative and provider-swappable — moving to a distributed JCache provider is a dependency +
  configuration change, not a code change, because the read path only uses `@Cacheable`
  (`VetRepository.java:44-56`).
- ✅ Observable — JCache statistics are enabled and surfaced via JMX (`CacheConfiguration.java:50`).

**Negative / trade-offs**
- ✅ The cache is per-JVM: with more than one replica each holds its own copy, so entries can be
  stale relative to each other and eviction is not coordinated. Not observable at `replicas: 1`
  (`k8s/petclinic.yml:22`) but a constraint on horizontal scaling.
- No explicit TTL/size limit is set through the JCache `MutableConfiguration`
  (`CacheConfiguration.java:49-51` notes size must be set via the provider); eviction relies on
  Caffeine/JCache defaults, so unbounded-growth behavior is provider-dependent (see Q2).
- Because no write path invalidates `vets`, the cache assumes vet data is externally static;
  out-of-band changes to the `vets`/`specialties` tables would not be reflected until eviction.

**Pattern relationship**
- **Instantiates** the catalog pattern *Read-Through In-Process Cache for Read-Mostly Data*
  (`docs/architecture-inventory/patterns/data/read-through-in-process-cache.md`, Candidate). No
  **Pattern Drift** — this ADR is the decision the pattern describes.
- **Pattern Update Proposal:** the "no invalidation path because the data is externally static"
  assumption is a reuse caveat worth noting in the pattern file, so future `@Cacheable` uses on
  mutable data don't copy it blindly.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Vet reads are `@Cacheable("vets")` | ✅ | `src/main/java/.../vet/VetRepository.java:44-56` |
| E2 | Caching enabled; `vets` cache created via JCache with stats | ✅ | `src/main/java/.../system/CacheConfiguration.java:31-51` |
| E3 | Cache provider is in-process Caffeine | ✅ | `pom.xml:82-83`; `build.gradle:45` |
| E4 | No networked cache (Redis/Memcached) dependency or manifest | ✅ | absence across `pom.xml`, `build.gradle`, `k8s/` |
| E5 | Single-replica deployment (per-JVM cache not yet observable) | ✅ | `k8s/petclinic.yml:22` (`replicas: 1`) |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Vet/specialty data is effectively static at runtime, so no cache-invalidation path is required. | The application exposes no write path that mutates vets; the cache has no eviction-on-write configured (`VetRepository.java`, `CacheConfiguration.java`). | If vets are edited (via a future feature or direct DB change), reads would serve stale data until natural eviction. | Confirm the vet-data lifecycle with the domain owner; add `@CacheEvict` if writes are introduced. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | At what replica count / coherency requirement should the cache move to a distributed provider? | The in-process choice is a scaling constraint; the JCache abstraction makes the switch cheap but it is undecided. | Revisit when replicas > 1 and cross-instance staleness becomes material. | Platform architecture owner |
| Q2 | Are the effective Caffeine size/TTL defaults acceptable, or should explicit bounds be set? | Unbounded or provider-default eviction affects memory footprint under load. | Set an explicit size/TTL if the `vets` dataset grows or memory pressure appears. | Platform architecture owner |
