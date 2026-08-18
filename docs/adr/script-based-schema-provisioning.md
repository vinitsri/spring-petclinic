# ADR: Script-Based Schema Provisioning without a Migration Tool

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-004 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Storage · **Impact:** high · **Dependencies:** none

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

Database schema and seed data are applied from **checked-in, per-engine SQL scripts** at application
startup, with Hibernate's runtime DDL generation disabled.

- ✅ `spring.jpa.hibernate.ddl-auto=none` — Hibernate never creates or mutates the schema at runtime
  (`src/main/resources/application.properties:10`).
- ✅ Schema and data are loaded via Spring SQL init from per-engine locations —
  `spring.sql.init.schema-locations=classpath*:db/${database}/schema.sql` and
  `...data-locations=classpath*:db/${database}/data.sql` (`application.properties:2–4`), resolving to
  `src/main/resources/db/{h2,mysql,postgres}/{schema,data}.sql`.
- ✅ There is **no** Flyway or Liquibase dependency in `pom.xml` or `build.gradle`, and no versioned
  migration directory anywhere in the tree (architecture-baseline §3).

The consequence is that schema evolution has no version history, no forward/rollback migration
records, and three hand-maintained SQL variants that must be kept in sync (a drift risk). This is a
durable data-lifecycle decision that governs how every future schema change is delivered.

## Decision

**Provision the relational schema and seed data from checked-in per-engine SQL scripts executed at
startup (`spring.sql.init`), with `spring.jpa.hibernate.ddl-auto=none`** — rather than adopting a
versioned migration tool (Flyway/Liquibase) or relying on Hibernate auto-DDL.

Schema changes are made by editing `db/{h2,mysql,postgres}/schema.sql` (and `data.sql` for seed
data); the running application never alters the schema itself.

## Alternatives Considered

1. **Versioned migrations via Flyway or Liquibase.**
   *Rejected (for the current state)* — adds a dependency and migration-authoring workflow. For a
   sample/reference application whose schema is small and rarely changes, the ceremony was judged not
   to pay for itself. The cost of this rejection is explicit and acknowledged: no migration history,
   no repeatable rollback, and manual cross-engine synchronization. 🎯 This is the recommended
   evolution path the moment the schema is under continuous change in a shared/production environment
   (see Open Questions).

2. **Hibernate auto-DDL (`ddl-auto=update` / `create`).**
   *Rejected* — hands schema authority to the ORM's diffing heuristics, which are non-deterministic
   across dialects, unsafe for data-bearing environments (silent/partial changes), and cannot express
   engine-specific objects like the PostgreSQL functional unique index. Explicit `ddl-auto=none`
   keeps the schema authoritative and reviewable in source control.

3. **A single canonical SQL script targeting one engine.**
   *Rejected* — would break the multi-engine portability the platform relies on (H2/MySQL/PostgreSQL
   selected by profile); some objects (e.g. the PostgreSQL `LOWER(name)` functional unique index)
   have no exact H2 equivalent and must be expressed per engine.

## Consequences

**Positive**
- ✅ Schema is fully versioned in source control, human-reviewable, and deterministic — the running
  app cannot silently mutate it (`ddl-auto=none`).
- Zero extra runtime/build dependency; startup provisioning is built into Spring Boot.
- Per-engine scripts allow engine-specific objects to be expressed precisely.

**Negative / trade-offs**
- ✅ **No migration history or rollback** — there is no record of applied versions and no
  down-migration mechanism.
- ✅ **Drift risk across three hand-maintained variants** — `db/h2`, `db/mysql`, `db/postgres` must
  be edited in lockstep; a missed edit diverges engines silently.
- Startup scripts are typically full re-creation (idempotency depends on script content), which is
  suited to ephemeral/dev databases more than long-lived production data.

**Pattern relationship**
- **Instantiates** *Script-Based Relational Schema Provisioning*
  (`docs/architecture-inventory/patterns/data/script-based-schema-provisioning.md`, Candidate) and
  is tightly coupled to *Profile-Based Environment & Datastore Selection* (the per-engine `${database}`
  indirection). No **Pattern Drift**.
- **Pattern Update Proposal:** if a migration tool is later adopted, this ADR should be superseded and
  the pattern deprecated in favor of a new "versioned migrations" pattern.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Hibernate runtime DDL disabled | ✅ | `src/main/resources/application.properties:10` (`spring.jpa.hibernate.ddl-auto=none`) |
| E2 | Schema/data loaded from per-engine scripts via `spring.sql.init` | ✅ | `application.properties:2–4`; `src/main/resources/db/{h2,mysql,postgres}/{schema,data}.sql` |
| E3 | No Flyway/Liquibase dependency; no migration directory | ✅ | `pom.xml`, `build.gradle` (absent); architecture-baseline §3 |
| E4 | Engine-specific objects require per-engine scripts | ✅ | `db/postgres/schema.sql` functional unique index vs `db/h2/schema.sql` (architecture-baseline §3) |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Startup SQL init runs in every environment (not disabled per-profile). | `spring.sql.init` locations are set in the base `application.properties` with no observed `mode=never` override. | If disabled in some profile, that environment would start with no schema. | Check `spring.sql.init.mode` across `application-*.properties` and per-environment config. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR and the schema-change process. | Platform architecture / data owner | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | At what point should a versioned migration tool (Flyway/Liquibase) replace startup SQL scripts? | Without migrations there is no rollback or history once the schema is under continuous change in a shared/production DB. | Adopt migrations when the schema is data-bearing and changes regularly; supersede this ADR then. | Platform / data owner |
| Q2 | How is cross-engine schema parity verified today? | Three hand-maintained variants can drift silently. | — (no automated parity check observed). | Platform / data owner |
