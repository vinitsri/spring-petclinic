# ADR: Layered (Defense-in-Depth) Uniqueness Enforcement

- **Status:** Accepted (retroactive — documents an existing, evidenced decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture / owner-domain team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-015 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Storage · **Impact:** low · **Dependencies:** ADR-CANDIDATE-003, ADR-CANDIDATE-005

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The business invariant "an owner cannot have two pets with the same name" is enforced at **two
layers** — an application pre-check and a database constraint — with the resulting DB error
translated back into a user-facing field error.

- ✅ **Application pre-check:** on create, `PetController.processCreationForm` rejects a duplicate
  before persisting — `owner.getPet(pet.getName(), true) != null` →
  `result.rejectValue("name", "duplicate", "already exists")`
  (`src/main/java/org/springframework/samples/petclinic/owner/PetController.java:110-113`). The
  `PetValidator` separately enforces name *presence* (`PetValidator.java:39-42`).
- ✅ **Database constraint:** the schema enforces the same rule independently. PostgreSQL uses a
  **functional unique index** on `(owner_id, LOWER(name))` — case-insensitive —
  (`src/main/resources/db/postgres/schema.sql:45`), while H2 and MySQL use a plain composite
  `UNIQUE (owner_id, name)` (`db/h2/schema.sql:55`; `db/mysql/schema.sql:47`). This per-engine
  divergence is a direct consequence of the multi-engine decision
  ([multi-engine-datastore-spring-profiles](multi-engine-datastore-spring-profiles.md)).
- ✅ **Error translation:** writes go through the aggregate root
  ([aggregate-root-persistence-boundary](aggregate-root-persistence-boundary.md)) via
  `owners.saveAndFlush(owner)`; a `DataIntegrityViolationException` is caught, matched against the
  named constraint (`isDuplicatePetNameViolation` tests the message for `unique_owner_pet_name`,
  `PetController.java:202-205`), and re-surfaced as the same `name`/`duplicate` field error
  (`PetController.java:128-133,170-174`) rather than a 500.

The two layers cover each other's weaknesses: the app pre-check gives a clean UX on the common case,
and the DB constraint closes the check-then-act race the pre-check alone cannot (two concurrent
requests both pass the in-memory check, then one loses at commit and is translated to a field error).

## Decision

**Enforce the "one pet name per owner" uniqueness invariant in depth: an application-layer pre-check
for good UX, backed by a database unique constraint as the authoritative guarantee, with the
`DataIntegrityViolationException` translated into the same field-level validation error.** Neither
layer is removed in favor of the other; the DB constraint is the source of truth for correctness and
the app pre-check is the source of a friendly message. This convention is the reusable template for
enforcing cross-engine invariants in the platform.

## Alternatives Considered

1. **Application-layer enforcement only (no DB constraint).**
   *Rejected* — the in-memory `owner.getPet(...)` check (`PetController.java:110`) is a
   check-then-act with a race window: two concurrent create requests can both see "no duplicate" and
   both insert. Without the DB unique index (`db/postgres/schema.sql:45`) the invariant would be
   violable under concurrency, so app-only enforcement cannot guarantee correctness.

2. **Database constraint only (no app pre-check, surface the raw error).**
   *Rejected* — a bare DB constraint guarantees correctness but yields a poor experience: without the
   pre-check and the `isDuplicatePetNameViolation` translation (`PetController.java:202-205`), the
   user would get a generic integrity-violation/500 instead of an inline "already exists" on the
   `name` field. The translation layer exists precisely to avoid that.

3. **Optimistic UI with no server-side uniqueness (rely on client validation).**
   *Rejected outright* — client-side checks are trivially bypassable and provide no data-integrity
   guarantee. The server is the only trustworthy enforcement point, so this was never viable for a
   correctness invariant.

## Consequences

**Positive**
- ✅ The invariant is **race-safe**: even if two requests pass the app pre-check, the DB unique
  constraint rejects the second and the app converts it to a clean field error
  (`PetController.java:128-133,202-205`).
- ✅ Common-case UX is clean: duplicates are usually caught by the pre-check before any DB round-trip
  (`PetController.java:110-113`).
- Provides a **reusable convention** for future invariants: pre-check + named DB constraint +
  exception translation keyed on the constraint name.

**Negative / trade-offs**
- ✅ **Per-engine semantic divergence:** PostgreSQL enforces case-insensitive uniqueness via
  `LOWER(name)` (`db/postgres/schema.sql:45`), whereas H2/MySQL enforce case-sensitive uniqueness
  (`db/h2/schema.sql:55`; `db/mysql/schema.sql:47`). "Fluffy" vs "fluffy" is a duplicate on Postgres
  but allowed on H2/MySQL — the same input can behave differently across profiles.
- ✅ **Constraint-name coupling:** the translation matches the literal string `unique_owner_pet_name`
  in the exception message (`PetController.java:204`); renaming the constraint in any `schema.sql`
  silently breaks the error translation (the write would then throw a raw 500).
- The rule is expressed in two places (code + three SQL files), so a change to the invariant must be
  made consistently in all of them.

**Pattern relationship**
- **Instantiates** the *Layered Uniqueness Enforcement* pattern
  (`patterns/data/layered-uniqueness-enforcement.md`, Candidate) — this ADR is its governing
  decision. It also composes with the *Aggregate-Root Persistence* pattern (writes flow through
  `OwnerRepository`) and the *Profile-Based Environment & Datastore Selection* pattern (per-engine
  constraint expression). **No Pattern Drift.**
- **Pattern Update Proposal:** the pattern file should flag two reusable cautions — (a) keep the DB
  constraint **name** stable because error translation depends on it, and (b) align case-sensitivity
  semantics across engines (see Q1) — so future invariants avoid the same traps.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | App-layer duplicate pre-check rejects before persist | ✅ | `owner/PetController.java:110-113` |
| E2 | Writes flow through the Owner aggregate via `saveAndFlush` | ✅ | `owner/PetController.java:124-133` |
| E3 | PostgreSQL functional unique index on `(owner_id, LOWER(name))` (case-insensitive) | ✅ | `src/main/resources/db/postgres/schema.sql:45` |
| E4 | H2/MySQL composite `UNIQUE (owner_id, name)` (case-sensitive) | ✅ | `db/h2/schema.sql:55`; `db/mysql/schema.sql:47` |
| E5 | `DataIntegrityViolationException` matched on constraint name and translated to a field error | ✅ | `owner/PetController.java:128-133,170-174,202-205` |
| E6 | `PetValidator` enforces name presence (distinct from uniqueness) | ✅ | `owner/PetValidator.java:39-42` |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The case-sensitivity divergence (Postgres case-insensitive vs H2/MySQL case-sensitive) is incidental, not an intended per-engine rule. | Only Postgres uses `LOWER(name)`; H2/MySQL use a plain composite unique with no stated rationale. | If case-insensitivity is a required business rule, H2/MySQL are non-compliant and duplicates leak on those engines. | Confirm the intended uniqueness semantics with the owner-domain owner; align schemas. |
| A2 | The production engine is PostgreSQL, so the case-insensitive index is the one that governs real data. | `k8s/petclinic.yml:35-36` pins `SPRING_PROFILES_ACTIVE=postgres`. | If another engine is used in production, the weaker (case-sensitive) rule applies. | Confirm the production engine with the deployment owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a decision owner to this ADR and to Q1. | Platform architecture / owner-domain team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Should uniqueness be case-insensitive on all engines (align H2/MySQL to Postgres)? | Today the same input can be a duplicate on Postgres but allowed on H2/MySQL, an inconsistent invariant. | 🎯 Normalise to case-insensitive across engines (e.g. functional/generated-column index) so behavior is engine-independent. | Owner-domain / data owner |
| Q2 | Should error translation stop depending on the literal constraint name `unique_owner_pet_name`? | A schema rename silently converts a friendly field error into a raw 500. | Centralise the constraint-name constant so schema and code cannot drift. | Owner-domain owner |
