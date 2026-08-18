# ADR: Build-Time Quality & Supply-Chain Gates as Governance

- **Status:** Accepted (retroactive — documents an existing, evidenced decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture / build team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-013 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Other (Governance) · **Impact:** medium · **Dependencies:** none

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The repository has **no ADRs, no `docs/decisions/`, and no separate policy engine** (verified in
architecture-baseline §7 and reaffirmed by this discovery run). In their absence, the platform's
governance is enforced **mechanically through the build**: a fixed set of quality and supply-chain
checks run during the build and fail it on violation.

- ✅ **Code style / format:** `spring-javaformat-maven-plugin` runs a `validate` goal bound to the
  `validate` phase (`pom.xml:207`), and Gradle applies `io.spring.javaformat` (`build.gradle:8`).
- ✅ **Static analysis:** `maven-checkstyle-plugin` with Checkstyle `12.3.1`
  (`pom.xml:220-226`, version pinned at `pom.xml:31,35`); Gradle applies the `checkstyle` plugin
  (`build.gradle:3,57-58,70-84`).
- ✅ **No-plaintext-HTTP gate:** `nohttp-checkstyle` runs a `check` goal in the `validate` phase
  (`pom.xml:229-246`), enforced identically in Gradle (`build.gradle:9,75-84`).
- ✅ **Toolchain enforcement:** `maven-enforcer-plugin` fails the build if the JDK is below the
  required Java version (`pom.xml:187-203`, `requireJavaVersion`).
- ✅ **Supply-chain SBOM:** `cyclonedx-maven-plugin` generates a CycloneDX SBOM (`pom.xml:308-312`),
  and Gradle applies `org.cyclonedx.bom` (`build.gradle:7,65`). The SBOM is surfaced by Actuator when
  present on the classpath (`pom.xml:306-307` comment).
- ✅ Both CI workflows run the full build — `./mvnw -B verify` and `./gradlew build` — so every
  push/PR to `main` executes these gates (`.github/workflows/maven-build.yml`,
  `.github/workflows/gradle-build.yml`).

Because these checks are bound to build phases and run in CI, a violation **fails the build** rather
than producing an advisory warning — the gates are load-bearing, not decorative.

## Decision

**Adopt build-time gates — spring-javaformat, Checkstyle, nohttp, the Maven enforcer, and CycloneDX
SBOM generation — as the platform's governance mechanism, wired into the build lifecycle and CI so a
violation fails the build.** Governance is enforced by the toolchain, not by review convention or an
external policy service. Contributions cannot merge green without satisfying every gate; the SBOM is
produced on every build for supply-chain visibility.

## Alternatives Considered

1. **Review-convention governance (human PR review, no automated gates).**
   *Rejected* — relying on reviewers to catch style, formatting, insecure-HTTP, and toolchain drift
   is inconsistent and unenforceable; it also produces no SBOM. The repository deliberately encodes
   these as failing build steps (`pom.xml:187,207,220,308`) precisely so compliance does not depend
   on a reviewer noticing. Automated gates give deterministic enforcement that review cannot.

2. **A dedicated external policy engine (e.g. OPA/Conftest, a separate security-scan service).**
   *Rejected as over-engineered for this platform* — a standalone policy engine adds a service to
   run and maintain and a second place governance lives. For a single deployable, binding the checks
   to the existing Maven/Gradle lifecycle keeps governance co-located with the code it governs, at
   zero extra infrastructure. A policy engine would be warranted only for cross-repo, org-wide
   policy that this single-repo platform does not have.

3. **Advisory (non-failing) checks — report but do not block.**
   *Rejected* — running the same tools in report-only mode preserves visibility while removing the
   guarantee. The value here is that the gates are **blocking** (bound to `validate`/`verify`), so
   the main branch is always format-clean, style-clean, http-clean, and SBOM-covered. Advisory mode
   would let violations accumulate.

## Consequences

**Positive**
- ✅ Style, formatting, static-analysis, no-HTTP, and JDK-version violations cannot merge green —
  every push/PR runs the gates in CI (`maven-build.yml`, `gradle-build.yml`).
- ✅ A CycloneDX SBOM is produced on every build and exposed via Actuator, giving continuous
  supply-chain inventory (`pom.xml:306-312`; `build.gradle:7,65`).
- Governance is versioned and reviewable in the same repository as the code, with no separate policy
  system to keep in sync.

**Negative / trade-offs**
- ✅ The gate set is **duplicated across both build systems** (`pom.xml` and `build.gradle`), so it
  inherits the sync risk documented in
  [dual-build-system-maven-gradle](dual-build-system-maven-gradle.md): a gate tightened in one build
  can be missed in the other.
- ❓ The gates cover **style/format/http/SBOM/JDK** but there is **no evidence of dependency
  vulnerability scanning** (e.g. OWASP dependency-check / CVE gate) beyond SBOM *generation* — SBOM
  inventory is not the same as a failing CVE gate (see Q1).
- Build time grows with each gate; the checks run on every CI invocation for both toolchains,
  compounding the dual-build cost.

**Pattern relationship**
- **Instantiates** the *Build-Time Quality & Supply-Chain Gates* pattern
  (`patterns/testing/build-time-quality-and-supply-chain-gates.md`, Candidate) — this ADR is the
  governing decision that would promote that pattern toward **Approved**. **No Pattern Drift.**
- **Pattern Update Proposal:** the pattern file should record that its gates must be maintained in
  **both** `pom.xml` and `build.gradle` until a canonical build is declared (ADR-CANDIDATE-010, Q1),
  and should flag the CVE-gate gap (Q1) as a candidate extension.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | spring-javaformat validates formatting in the `validate` phase | ✅ | `pom.xml:207`; `build.gradle:8` |
| E2 | Checkstyle static analysis pinned to v12.3.1 | ✅ | `pom.xml:31,35,220-226`; `build.gradle:3,57-58,70-84` |
| E3 | nohttp gate fails the build on plaintext HTTP references | ✅ | `pom.xml:229-246`; `build.gradle:9,75-84` |
| E4 | Maven enforcer requires a minimum JDK version | ✅ | `pom.xml:187-203` |
| E5 | CycloneDX SBOM generated on every build | ✅ | `pom.xml:308-312`; `build.gradle:7,65` |
| E6 | All gates run in CI on push/PR to main | ✅ | `.github/workflows/maven-build.yml`; `.github/workflows/gradle-build.yml` |
| E7 | No ADRs / decision register exist, so the build is the governance mechanism | ✅ | absence of `docs/adr/` prior artifacts and `docs/decisions/`; architecture-baseline §7 |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The gates are treated as blocking (build-failing), not advisory. | They are bound to `validate`/`verify` phases and run inside `verify`/`build` in CI. | If CI ignores gate failures, governance is not actually enforced and this ADR overstates it. | Confirm CI fails the job on gate violation with the build owner; inspect a failing run. |
| A2 | SBOM generation is the extent of the intended supply-chain posture (no CVE-blocking gate expected). | Only CycloneDX generation is present; no vulnerability-scan plugin was found. | If a CVE gate was expected, the posture has a gap that this ADR should call a target, not current state. | Confirm supply-chain scanning expectations with the security owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a decision owner to this ADR and to Q1. | Platform architecture / build team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Should a dependency-vulnerability (CVE) gate be added, given SBOM is generated but not scanned? | SBOM inventory without a failing CVE gate leaves known-vulnerable dependencies able to merge. | 🎯 Add an OWASP dependency-check (or equivalent) build-failing gate consuming the CycloneDX SBOM. | Platform security owner |
| Q2 | Once a canonical build is chosen (ADR-CANDIDATE-010), should the gate definitions live in one build only? | Removes the dual-maintenance sync risk for the gate set. | Consolidate gates onto the canonical build after the Maven-vs-Gradle decision. | Platform build owner |
