# ADR: First-Class GraalVM Native-Image Build Support

- **Status:** Accepted (retroactive — documents an existing, evidenced decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture / build team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-012 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Deployment · **Impact:** low · **Dependencies:** ADR-CANDIDATE-001

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The build carries **first-class GraalVM native-image support alongside the JVM build**, and the code
base actively maintains the reflection/resource hints that AOT compilation requires.

- ✅ The Maven build declares the GraalVM `native-maven-plugin` (`pom.xml:252-253`), and the Gradle
  build applies the `org.graalvm.buildtools.native` plugin (`build.gradle:6`). Both toolchains
  (see [dual-build-system-maven-gradle](dual-build-system-maven-gradle.md)) carry the native path.
- ✅ The application ships a hand-maintained `RuntimeHintsRegistrar` —
  `src/main/java/org/springframework/samples/petclinic/PetClinicRuntimeHints.java:25-36` — that
  registers resource patterns (`db/*`, `db/*/*`, `messages/*`, `mysql-default-conf`) and reflection/
  serialization hints for `BaseEntity`, `Person`, and `Vet`. These hints exist **specifically** so
  the closed-world native image can find resources and reflect over types that would otherwise be
  discovered dynamically on the JVM.
- ✅ The registered resource patterns mirror the schema-provisioning layout — the `db/*/*` pattern
  exists to bundle the per-engine `schema.sql`/`data.sql` scripts into the native image (relates to
  [script-based-schema-provisioning](script-based-schema-provisioning.md) and
  [multi-engine-datastore-spring-profiles](multi-engine-datastore-spring-profiles.md)).

The native path is a **second artifact type** for the same monolith (ADR-CANDIDATE-001): a
statically-compiled binary with fast startup and low memory footprint, produced from the same source
as the JVM jar.

## Decision

**Maintain first-class GraalVM native-image support as a supported build target alongside the JVM
build, backed by an explicitly maintained `PetClinicRuntimeHints` registrar that declares the
reflection and resource hints AOT compilation requires.** The JVM jar remains the default artifact;
the native image is an opt-in second artifact for startup-/footprint-sensitive deployments. Hint
maintenance is accepted as an ongoing cost paid to keep the native path building.

## Alternatives Considered

1. **JVM-only build (drop native support).**
   *Rejected* — dropping the `native-maven-plugin`/Gradle native plugin and deleting
   `PetClinicRuntimeHints` would remove all AOT-hint maintenance burden and shorten the reference
   build. It was not chosen because the native path is a deliberate capability of this reference
   application (fast cold start, low memory for scale-to-zero/serverless-style deployments), and the
   hints are already written and working. Retaining it costs only ongoing hint upkeep.

2. **Rely on runtime dynamic behavior with no explicit hints (auto-detection only).**
   *Rejected as unworkable for native* — GraalVM native image is closed-world: resources and
   reflection targets not discovered at build time fail at runtime. The seeded SQL under `db/**`,
   i18n bundles under `messages/**`, and JPA/serialization over the entity types are exactly the
   dynamic surfaces that break without explicit registration. `PetClinicRuntimeHints:29-35` exists
   because auto-detection is insufficient, so this alternative would produce a broken native image.

3. **Externalize hints via `reflect-config.json`/`resource-config.json` metadata files instead of a
   `RuntimeHintsRegistrar`.**
   *Rejected in favor of the code-based registrar* — hand-written GraalVM config JSON is an
   equivalent mechanism but is disconnected from the Java type system (no compiler check when a
   class is renamed/removed). The `RuntimeHintsRegistrar` approach references types directly
   (`BaseEntity.class`, `Person.class`, `Vet.class`), so refactors surface at compile time. This is
   the Spring-idiomatic choice and the one the code adopts.

## Consequences

**Positive**
- ✅ A low-startup, low-memory native artifact is buildable from the same source as the JVM jar, via
  either toolchain (`pom.xml:252-253`; `build.gradle:6`).
- ✅ AOT hints are code-checked against real types, so class refactors break the build rather than the
  native runtime (`PetClinicRuntimeHints.java:33-35`).

**Negative / trade-offs**
- ✅ **Hint maintenance is a standing obligation:** any new resource directory, reflected type, or
  serialization path must be added to `PetClinicRuntimeHints` or the native image silently misbehaves
  at runtime. New per-engine SQL directories are covered by the `db/*/*` pattern
  (`PetClinicRuntimeHints.java:29-30`), but new resource roots are not.
- ❓ Native builds are substantially slower and more memory-hungry than JVM builds; whether CI
  actually exercises the native profile on every change (vs. JVM-only) is not evidenced in the
  workflows read (`.github/workflows/**`) — see Q1.
- The native path doubles the test surface: a change can pass on the JVM and still fail natively if
  hints are missing, so native correctness needs its own verification to be trustworthy.

**Pattern relationship**
- **No dedicated catalog pattern** exists for native-image support in
  `docs/architecture-inventory/patterns/`. The registrar's resource patterns are the AOT counterpart
  of the *Script-Based Relational Schema Provisioning* pattern
  (`patterns/data/script-based-schema-provisioning.md`) — the same `db/**` scripts must be bundled
  into the closed-world image. **No Pattern Drift.**
- **Pattern Update Proposal:** add a *GraalVM native-image hints maintenance* pattern noting that any
  new dynamic resource/reflection surface must be registered in `PetClinicRuntimeHints`, so future
  features do not break the native artifact.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Maven declares the GraalVM native-image plugin | ✅ | `pom.xml:252-253` |
| E2 | Gradle applies the GraalVM native buildtools plugin | ✅ | `build.gradle:6` |
| E3 | A maintained RuntimeHintsRegistrar declares AOT resource/reflection hints | ✅ | `src/main/java/org/springframework/samples/petclinic/PetClinicRuntimeHints.java:25-36` |
| E4 | Resource hints bundle the per-engine SQL and i18n bundles into the native image | ✅ | `PetClinicRuntimeHints.java:29-32` |
| E5 | Reflection/serialization hints registered for core entity types | ✅ | `PetClinicRuntimeHints.java:33-35` |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The native image is a supported, intended artifact rather than dead configuration. | Both build files carry the native plugin and the hints registrar is actively maintained with entity-specific registrations. | If native is unused/abandoned, the hint-maintenance cost is unjustified and the config should be removed. | Confirm with the build owner whether native images are built/released. |
| A2 | The JVM jar is the default/primary artifact and native is opt-in. | Application config and k8s manifests target the JVM runtime; no native artifact is referenced in `k8s/*.yml`. | If native is the release target, deployment ADRs (011) understate its role. | Confirm the release artifact type with the deployment owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a decision owner to this ADR and to Q1. | Platform architecture / build team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Does CI build and test the native image on every change, or only the JVM jar? | If native is not CI-exercised, missing hints will surface only at deploy time, undermining the "supported target" claim. | Add a scheduled/gated native build so hint regressions are caught early. | Platform build owner |
| Q2 | Is the native image ever the released artifact, or purely a demonstration capability? | Determines whether native correctness is release-critical or best-effort. | — | Platform build/deployment owner |
