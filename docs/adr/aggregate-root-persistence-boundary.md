# ADR: Aggregate-Root Persistence Boundary (Owner Aggregate)

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-003 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Storage · **Impact:** high · **Dependencies:** [single-deployable-layered-monolith](single-deployable-layered-monolith.md) (ADR-CANDIDATE-001)

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

Pets and visits are persisted **only** through the `Owner` aggregate root and its repository, not
through independent `Pet` or `Visit` repositories.

- ✅ `Owner` owns its pets via `@OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)`
  (`owner/Owner.java:64`), and exposes aggregate operations `addPet(...)` and
  `addVisit(petId, visit)` (`owner/Owner.java:93–173`).
- ✅ `OwnerRepository extends JpaRepository<Owner, Integer>` (`owner/OwnerRepository.java:36`) is the
  write path; there is no `PetRepository`/`VisitRepository` performing writes.
- ✅ `PetController` mutates the aggregate in memory (`owner.addPet(pet)`) and persists it via
  `this.owners.saveAndFlush(owner)` (`owner/PetController.java:103,125–126,197–199`); the same
  aggregate-root save path carries pet and visit changes.

This is a deliberate Domain-Driven-Design consistency-boundary decision: the `Owner` aggregate is the
transactional unit, and all child mutations flow through it. It has lasting impact on transaction
scope, the shape of write APIs, and how future domains are modeled. It is bounded by the
single-deployable monolith (ADR-CANDIDATE-001), which keeps the aggregate and its schema in one JVM.

## Decision

**Persist pets and visits exclusively through the `Owner` aggregate root's repository
(`OwnerRepository.save`/`saveAndFlush`), treating `Owner` as the DDD consistency boundary** — rather
than exposing `Pet` and `Visit` as independent aggregates each with its own repository. Child entities
are added to the in-memory `Owner` graph and cascaded to the database in a single aggregate-root save.

## Alternatives Considered

1. **A repository per entity (`PetRepository`, `VisitRepository`) exposing pets/visits as independent
   aggregates.**
   *Rejected* — yields a flatter CRUD surface but weakens the consistency boundary: pets/visits could
   be created or mutated without their owner context, splitting what is conceptually one transactional
   unit and inviting orphan/ownership-integrity bugs. Routing writes through the owner keeps the
   invariant (a pet belongs to exactly one owner) enforced at the aggregate boundary.

2. **A transactional service layer over per-entity repositories that reconstructs the boundary in
   application code.**
   *Rejected* — reintroduces the same consistency boundary the aggregate already provides, but as
   hand-written orchestration that must be applied consistently everywhere, rather than as a
   structural property of the model. More code, more ways to get it wrong.

3. **Direct entity persistence via `EntityManager` without the aggregate abstraction.**
   *Rejected* — bypasses the repository abstraction and cascade semantics, scattering persistence
   concerns across controllers and losing the single, reviewable write path.

## Consequences

**Positive**
- ✅ A single, clear write path per aggregate; child mutations cascade atomically with the root save
  (`CascadeType.ALL`, `saveAndFlush`), keeping the transaction scope well-defined.
- The owner→pet→visit invariant is enforced structurally, not by convention.
- New domains have a clear precedent for modeling consistency boundaries.

**Negative / trade-offs**
- ✅ `FetchType.EAGER` on the owner→pets association loads the full pet graph on every owner read,
  which does not scale for owners with many pets (`owner/Owner.java:64`).
- No direct pet/visit read/write API — consumers must go through the owner, which is less convenient
  for narrow child-only queries.
- Couples C1/C2/C3 (owners, pets, visits) in shared code and data ownership, reinforcing the coarse
  module boundary noted in ADR-CANDIDATE-001.

**Pattern relationship**
- **Instantiates** *Aggregate-Root Persistence via Spring Data JPA*
  (`docs/architecture-inventory/patterns/data/aggregate-root-persistence.md`, Candidate) and is the
  write path relied on by *Layered Uniqueness Enforcement* (the duplicate-pet-name guard runs on the
  aggregate save — candidate ADR-CANDIDATE-015). No **Pattern Drift**.
- **Pattern Update Proposal:** none new; promote the pattern to Approved alongside this ADR so future
  per-entity-repository writes are flagged as drift.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | `Owner` owns pets via cascading `@OneToMany` and aggregate methods | ✅ | `owner/Owner.java:64` (`cascade = CascadeType.ALL`, `EAGER`), `:93–173` (`addPet`, `addVisit`) |
| E2 | `OwnerRepository` is the JPA write path; no pet/visit write repos | ✅ | `owner/OwnerRepository.java:36` (`extends JpaRepository<Owner, Integer>`) |
| E3 | Child writes go through the aggregate root save | ✅ | `owner/PetController.java:103,125–126,197–199` (`owner.addPet(...)` → `owners.saveAndFlush(owner)`) |
| E4 | Aggregate save path anchors the uniqueness guard | ✅ | `owner/PetController.java:128–129,202` (`DataIntegrityViolationException` → `isDuplicatePetNameViolation`) |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Owners are expected to have a small number of pets, so `FetchType.EAGER` on the pets association is acceptable. | Reference veterinary-clinic domain; seed data has few pets per owner. | At scale, eager loading of the pet graph on every owner read becomes a performance problem. | Load-test owner reads with realistic pet counts; consider `LAZY` + fetch joins if needed. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR. | Platform architecture / data owner | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Should the owner→pets association move from `EAGER` to `LAZY` fetching? | Eager fetch loads the full pet graph on every owner read and does not scale. | 🎯 Move to `LAZY` with explicit fetch joins where the pet list is needed, if read volume grows. | Platform / data owner |
