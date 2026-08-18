# ADR: Multi-Engine Datastore via Spring Profiles (H2 / MySQL / PostgreSQL)

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-005 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Storage · **Impact:** medium · **Dependencies:** ADR-CANDIDATE-004 ([script-based-schema-provisioning](script-based-schema-provisioning.md))

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The Petclinic platform is a single deployable (see
[single-deployable-layered-monolith](single-deployable-layered-monolith.md)) that must run against
more than one relational engine without recompilation. The engine is chosen at deploy time by
activating a named Spring profile.

- ✅ The default profile runs against **H2 in-memory** (`spring.sql.init` picks the engine via the
  `database` property, default `h2`) — `src/main/resources/application.properties:2-4`.
- ✅ A **MySQL** profile overrides `database=mysql` and supplies a JDBC URL / credentials from
  environment variables — `src/main/resources/application-mysql.properties:2-7`
  (`jdbc:mysql://localhost/petclinic`, `spring.sql.init.mode=always`).
- ✅ A **PostgreSQL** profile overrides `database=postgres` with its own JDBC URL / credentials —
  `src/main/resources/application-postgres.properties:2-7`.
- ✅ The Kubernetes deployment activates PostgreSQL at runtime via `SPRING_PROFILES_ACTIVE=postgres`
  — `k8s/petclinic.yml:35-36`.
- ✅ Local databases are provisioned per engine by `docker-compose.yml` (`mysql:9.7`,
  `postgres:18.4`, each service named after its profile).
- ✅ The `database` property drives which per-engine SQL directory is applied at startup, so three
  parallel schema variants exist — `src/main/resources/db/{h2,mysql,postgres}/schema.sql`.

Supporting three engines is a portability decision with a standing maintenance cost: schema and seed
scripts must be kept in sync across three dialects, and engine-specific features cannot be used
freely because the lowest-common-denominator across engines constrains the schema. A concrete
example of the resulting divergence is the "one pet name per owner" uniqueness rule, expressed as a
functional unique index `unique_owner_pet_name ON pets (owner_id, LOWER(name))` in PostgreSQL
(`src/main/resources/db/postgres/schema.sql:45`) but as a plain `UNIQUE (owner_id, name)` constraint
in H2 (`src/main/resources/db/h2/schema.sql:55`) — the two are not semantically identical
(case-insensitivity differs).

## Decision

**Support H2, MySQL, and PostgreSQL from one build artifact, selecting the active engine at deploy
time through a named Spring profile** (`SPRING_PROFILES_ACTIVE` / the `database` property), with a
dedicated `application-<engine>.properties` and a dedicated `db/<engine>/` script set per engine.
H2 in-memory remains the zero-configuration default for development and tests; PostgreSQL is the
engine activated in the Kubernetes deployment.

## Alternatives Considered

1. **Commit to a single relational engine (e.g. PostgreSQL only).**
   *Rejected* — would eliminate the three-way schema maintenance burden and allow free use of
   engine-specific features, but it removes the frictionless H2 default that lets the app and its
   integration tests run with no external database, and it removes the demonstration value of
   engine portability that this reference application is expected to show. The cost of that lost
   developer ergonomics outweighs the schema-maintenance saving at this scale.

2. **Abstract the dialect differences behind a schema-management/ORM DDL layer (Hibernate
   auto-DDL, or a dialect-aware migration tool generating per-engine DDL).**
   *Rejected here* — Hibernate `ddl-auto` is deliberately disabled
   (`application.properties:10`, see [script-based-schema-provisioning](script-based-schema-provisioning.md)),
   and no migration tool is present, so generated cross-dialect DDL is out of scope. Auto-DDL would
   also forfeit the hand-tuned per-engine constructs (e.g. the PostgreSQL functional unique index)
   that the checked-in scripts rely on.

3. **Run different engines per environment but as separate build artifacts / branches.**
   *Rejected* — multiplies build and release artifacts and defeats the "one image, many
   backends" property that makes the deployable simple to promote across environments with only an
   environment-variable change (`SPRING_PROFILES_ACTIVE`).

## Consequences

**Positive**
- ✅ One image runs against any of the three engines with only an environment change
  (`k8s/petclinic.yml:35-36`); no rebuild is required to switch backends.
- ✅ Zero-configuration local/test runs on the default H2 profile — no external database needed
  (`application.properties:2`).
- Credentials and URLs are externalized to environment variables per engine
  (`application-mysql.properties`, `application-postgres.properties`), keeping secrets out of the
  image.

**Negative / trade-offs**
- ✅ Three `schema.sql` (and `data.sql`) variants must be kept in sync by hand
  (`db/{h2,mysql,postgres}/`); drift between them is a real risk (compounded by the absence of a
  migration tool — [script-based-schema-provisioning](script-based-schema-provisioning.md)).
- ✅ Engine-specific optimizations are constrained by the need to express each rule across all
  three dialects; the `unique_owner_pet_name` divergence (case-insensitive in PostgreSQL, plain in
  H2) shows the semantic gap this creates.
- Testing burden multiplies: correctness must be validated on more than one engine to trust the
  portability claim.

**Pattern relationship**
- **Instantiates** the catalog pattern *Profile-Based Environment & Datastore Selection*
  (`docs/architecture-inventory/patterns/deployment/profile-based-environment-configuration.md`,
  Candidate) and depends on *Script-Based Relational Schema Provisioning*
  (`patterns/data/script-based-schema-provisioning.md`). No **Pattern Drift** — this ADR is the
  decision those patterns describe.
- **Pattern Update Proposal:** the cross-dialect divergence in `unique_owner_pet_name` is a reusable
  caution worth capturing in the profile-based pattern file (portability constrains constraint
  expression). Relates to [layered uniqueness enforcement — ADR-CANDIDATE-015, backlog].

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Engine selected by the `database` property; H2 is the default | ✅ | `src/main/resources/application.properties:2-4` |
| E2 | MySQL profile overrides engine + externalizes URL/credentials | ✅ | `src/main/resources/application-mysql.properties:2-7` |
| E3 | PostgreSQL profile overrides engine + externalizes URL/credentials | ✅ | `src/main/resources/application-postgres.properties:2-7` |
| E4 | Kubernetes activates the `postgres` profile at runtime | ✅ | `k8s/petclinic.yml:35-36` |
| E5 | Three per-engine script sets exist | ✅ | `src/main/resources/db/{h2,mysql,postgres}/{schema,data}.sql` |
| E6 | Per-engine constraint divergence (case-insensitive uniqueness) | ✅ | `db/postgres/schema.sql:45` vs `db/h2/schema.sql:55` |
| E7 | Local per-engine databases provisioned by compose (`mysql:9.7`, `postgres:18.4`) | ✅ | `docker-compose.yml` |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | PostgreSQL is the intended production engine because it is the only one activated in the deploy manifest. | `k8s/petclinic.yml:35-36` sets `SPRING_PROFILES_ACTIVE=postgres`; H2/MySQL are dev/test conveniences. | If production actually runs MySQL (or H2), the tuned PostgreSQL constructs would not apply and the drift risk shifts. | Confirm the production engine with the platform/ops owner. |
| A2 | The three schema variants are intended to be semantically equivalent aside from documented dialect differences. | They model the same domain; divergences observed are engine-capability driven. | Undocumented divergence would cause environment-specific behavior (e.g. the case-insensitivity gap in `unique_owner_pet_name`). | Diff the three `schema.sql` files and reconcile intended vs incidental differences. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Should the three schema variants be generated from one canonical source (or a migration tool) to remove hand-sync drift? | Determines whether multi-engine support keeps its current maintenance cost or is consolidated. | Revisit jointly with the schema-provisioning decision (ADR-CANDIDATE-004) if drift incidents appear. | Platform data owner |
| Q2 | Is MySQL a supported production target or only a demonstration/test path? | Affects how much validation effort the MySQL schema/profile warrants. | Treat as test/demo unless declared otherwise. | Platform architecture owner |
