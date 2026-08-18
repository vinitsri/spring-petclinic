# ADR: Dual Build System (Maven + Gradle) with Undeclared Canonical

- **Status:** Proposed (retroactive — documents an existing state with an unresolved decision point)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture / build team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-010 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Deployment · **Impact:** medium · **Dependencies:** none

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The repository maintains **two complete build systems** — Maven and Gradle — each wired into its own
CI workflow, with no in-repo statement of which is canonical for release.

- ✅ A Maven build (`pom.xml`) and a Gradle build (`build.gradle` + `settings.gradle`) both exist and
  both are functional (each declares the same dependency set, quality plugins, and native/SBOM
  tooling).
- ✅ Each has a dedicated CI workflow that triggers on push/PR to `main`:
  `.github/workflows/maven-build.yml:28` (`./mvnw -B verify`) and
  `.github/workflows/gradle-build.yml:30-31` (`./gradlew build`).
- ✅ The two builds duplicate the same governance tooling: Checkstyle, spring-javaformat, nohttp,
  and CycloneDX SBOM appear in **both** `pom.xml` (`:207,220,230,310`) and `build.gradle`
  (`:3,6-9,57-58,65`), plus GraalVM native support in both (`native-maven-plugin` `pom.xml:253`;
  `org.graalvm.buildtools.native` `build.gradle:6`).
- ✅ There is **no** document, property, or comment declaring which build is authoritative for
  producing the release artifact/image.

Supporting two build systems has an ongoing cost: dependency versions, plugin configuration, quality
gates, and SBOM/native settings must be kept in sync in two places, or the two builds drift and
produce subtly different artifacts. The offsetting benefit is contributor choice and reference value
(the sample demonstrates both toolchains).

## Decision

**Record that the platform maintains both a Maven and a Gradle build, each with its own CI workflow,
and that a single canonical build for release is NOT yet declared.** This ADR captures the existing
dual-build state and its trade-offs, and makes the *choice of canonical build* an explicit,
owner-assigned open decision (see Q1) rather than leaving it implicit. Until that decision is made,
both builds must be kept in sync by the contributor changing either one.

> This ADR is **Proposed**, not Accepted: the load-bearing question — which build governs release —
> is deliberately left open here and routed to the build owner, because the repository provides no
> evidence to decide it (Source-of-Truth rule: code shows two equal builds, so the ADR does not
> invent a winner).

## Alternatives Considered

1. **Declare one build system canonical (e.g. Maven for release) and keep the other as a
   convenience/CI check only.**
   *Preferred target (🎯) but not yet evidenced* — this removes the "which artifact is
   authoritative" ambiguity and lets SBOM/native/image automation hang off one source of truth. It
   is not recorded as the decision because nothing in the repo declares it; doing so would invent
   evidence. Captured as Q1.

2. **Remove one build system entirely (single toolchain).**
   *Rejected for now* — the cleanest way to eliminate the sync cost, but it discards the
   reference/demo value of showing both toolchains and would be a larger change than this
   documentation stage should force. Viable once a canonical build is chosen (Q1) and the team
   decides the second build's demo value no longer justifies its maintenance.

3. **Keep both builds and add automated parity enforcement (a CI check that fails if dependency
   sets / plugin configs diverge).**
   *Rejected as out of scope here* — a reasonable mitigation for the sync cost, but it adds
   tooling this ADR should not mandate before the canonical-build question is answered; noted as a
   possible follow-up if both builds are retained.

## Consequences

**Positive**
- ✅ Contributors can build with either toolchain; both are CI-validated on every push/PR
  (`maven-build.yml`, `gradle-build.yml`).
- The repository serves as a working reference for both Maven and Gradle configurations of the same
  application, including quality gates and native/SBOM tooling.

**Negative / trade-offs**
- ✅ Duplicated configuration must be kept in sync across `pom.xml` and `build.gradle` (dependencies,
  Checkstyle/format/nohttp, CycloneDX, native) — divergence risk on every dependency or plugin
  change.
- ✅ No declared canonical build means **release-artifact provenance is ambiguous**: it is
  undefined whether the released image/JAR comes from Maven or Gradle, which also leaves the
  buildpacks image path (`build-image` vs `bootBuildImage`) unresolved — see
  [cloud-native-buildpacks-image-build](cloud-native-buildpacks-image-build.md).
- Two CI workflows run per push, roughly doubling build minutes for the same validation.

**Pattern relationship**
- **No matching catalog pattern** exists for build-system topology in
  `docs/architecture-inventory/patterns/`. The *Build-Time Quality & Supply-Chain Gates* pattern
  (`patterns/testing/build-time-quality-and-supply-chain-gates.md`, Candidate — see
  ADR-CANDIDATE-013) is **duplicated across both** builds, which this ADR touches on. No **Pattern
  Drift**.
- **Pattern Update Proposal:** note in the quality-gates pattern that its gates must be maintained
  in *both* build files until a canonical build is declared, so future edits don't update only one.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Both Maven and Gradle builds exist and are functional | ✅ | `pom.xml`; `build.gradle`, `settings.gradle` |
| E2 | Each build has its own CI workflow on `main` push/PR | ✅ | `.github/workflows/maven-build.yml:28`; `.github/workflows/gradle-build.yml:30-31` |
| E3 | Quality/SBOM/native tooling duplicated across both builds | ✅ | `pom.xml:207,220,230,253,310`; `build.gradle:3,6-9,57-58,65` |
| E4 | No in-repo declaration of the canonical build for release | ✅ | absence of any such statement in `README.md`, `pom.xml`, `build.gradle`, workflows |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Both builds are intended to remain in sync (same dependencies, gates, and artifact) rather than diverging on purpose. | They currently mirror each other's dependency and plugin sets (`pom.xml` / `build.gradle`). | If they are intentionally different, "keep in sync" guidance is wrong and parity checks would false-alarm. | Confirm intent with the build owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR and to Q1. | Platform architecture / build team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Is Maven or Gradle the canonical build/release path? | Release-artifact provenance, SBOM/native automation, and the buildpacks image path all depend on a single source of truth. | — (unresolved; repo provides no basis to decide) | Platform build owner |
| Q2 | If both builds are retained, should CI enforce dependency/plugin parity between them? | Prevents silent drift between the two configurations on future changes. | Add a parity check only after Q1 is settled. | Platform build owner |
