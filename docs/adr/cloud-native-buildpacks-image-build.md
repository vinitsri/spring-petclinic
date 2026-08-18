# ADR: Cloud Native Buildpacks over a Hand-Written Dockerfile

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-009 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Deployment · **Impact:** medium · **Dependencies:** none

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The application is containerized for Kubernetes deployment, but the container image is produced by
**Cloud Native Buildpacks** through the Spring Boot build plugin rather than by a maintained
Dockerfile.

- ✅ The README documents image building via `spring-boot:build-image` and states explicitly
  *"There is no `Dockerfile` in this project"* — `README.md:45-52`.
- ✅ The Spring Boot Maven plugin is configured on the build (`spring-boot-maven-plugin`,
  `pom.xml:255-257`), and its `build-image` goal is the buildpacks entry point; the equivalent
  Gradle path is `bootBuildImage` (Spring Boot Gradle plugin).
- ✅ There is **no** `Dockerfile` (or `Containerfile`) anywhere in the tree — confirmed by directory
  search.
- ✅ The Kubernetes deployment references a pre-published image `dsyer/petclinic` —
  `k8s/petclinic.yml:33`.
- ❓ The pipeline that builds and publishes the specific `dsyer/petclinic` tag referenced by the
  manifest is **not in-repo** — the CI workflows build/verify (`maven-build.yml`, `gradle-build.yml`)
  and deploy manifests (`deploy-and-test-cluster.yml`) but none runs `build-image`/push. So the
  provenance of that exact published tag is unverifiable here.

Buildpacks trade fine-grained control of the base image and layers for reproducible,
maintenance-light OCI images (the builder handles the JRE, base OS, and layering, and can be rebased
to pick up base-image CVE fixes without a project rebuild).

## Decision

**Build the container image with Cloud Native Buildpacks via the Spring Boot build plugin
(`spring-boot:build-image` / `bootBuildImage`) and intentionally keep no Dockerfile in the
repository.** Base-image maintenance, layering, and OS/JRE provisioning are delegated to the
buildpacks builder rather than owned by a project-maintained Dockerfile.

## Alternatives Considered

1. **Maintain a hand-written Dockerfile.**
   *Rejected* — a Dockerfile gives full control of the base image, explicit layer ordering, and a
   project-owned CVE-patching cadence, but it makes the project responsible for base-image
   selection, JRE provisioning, and ongoing hardening/patching. Buildpacks provide reproducible,
   optimally-layered images and rebase-based CVE remediation with far less per-project maintenance,
   which suits a small application and its contributors. The README makes the "no Dockerfile" stance
   explicit (`README.md:45`).

2. **Jib (Google) or another Dockerfile-less image builder.**
   *Rejected* — Jib is a viable Dockerfile-free alternative, but the Spring Boot plugin already
   ships buildpacks support in-box (`spring-boot:build-image`), so adopting it needs no extra plugin
   and stays aligned with the framework's recommended packaging path. Adding Jib would introduce a
   second build-tooling dependency for no differentiating benefit here.

3. **Ship only a runnable JAR and let each environment build its own image.**
   *Rejected* — pushes image construction and its security posture onto every consumer, producing
   inconsistent images; a single buildpacks-produced image gives one reproducible artifact to
   promote across environments.

## Consequences

**Positive**
- ✅ No Dockerfile to maintain, review, or keep patched — image construction is delegated to the
  buildpacks builder (`README.md:45-52`).
- ✅ Reproducible, well-layered OCI images from a standard framework command
  (`spring-boot:build-image`, plugin at `pom.xml:255-257`); base-image CVE fixes can be picked up by
  rebasing rather than a full project rebuild.
- Consistent single image artifact to promote across environments (referenced as `dsyer/petclinic`,
  `k8s/petclinic.yml:33`).

**Negative / trade-offs**
- ✅ Less fine-grained control of the base image and layer composition than a hand-written
  Dockerfile; the builder dictates the base OS/JRE and update cadence.
- ❓ The build-and-publish pipeline for the deployed `dsyer/petclinic` tag is not in-repo
  (`k8s/petclinic.yml:33`; no `build-image` step in `.github/workflows/**`), so image provenance,
  builder version, and patch cadence for that tag cannot be verified from this repository.
- Buildpacks require a Docker daemon / builder to be available at build time (`README.md:45`),
  a constraint on where images can be produced.

**Pattern relationship**
- **No matching catalog pattern** exists in `docs/architecture-inventory/patterns/` for image
  packaging (the deployment category currently holds *Profile-Based Environment & Datastore
  Selection* only). No **Pattern Drift**.
- **Pattern Update Proposal:** add a *Buildpacks-Based Image Packaging (No Dockerfile)* deployment
  pattern so future services default to the same Dockerfile-less approach and are drift-checked
  against it. Relates to [dual-build-system-maven-gradle](dual-build-system-maven-gradle.md) (both
  build systems must expose an equivalent `build-image` path).

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Image built via buildpacks; "There is no Dockerfile" stated | ✅ | `README.md:45-52` |
| E2 | Spring Boot build plugin configured (buildpacks entry point) | ✅ | `pom.xml:255-257` |
| E3 | No Dockerfile/Containerfile in the tree | ✅ | directory search across the repo |
| E4 | Deployment references pre-published image `dsyer/petclinic` | ✅ | `k8s/petclinic.yml:33` |
| E5 | No `build-image`/push step in CI; published-tag provenance unverifiable | ❓ | `.github/workflows/{maven-build,gradle-build,deploy-and-test-cluster}.yml` |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The deployed `dsyer/petclinic` image is produced by the same buildpacks path documented in the README. | The README is the only documented build path and no Dockerfile exists; the tag name matches the sample's upstream author. | If the deployed tag is built differently (or is stale/third-party), the image's contents and patch cadence differ from what this ADR describes. | Confirm the publish pipeline/registry for `dsyer/petclinic` with the platform/ops owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |
| B2 | The image build-and-publish pipeline is not in this repository. | Verifying image provenance, builder version, and CVE-patch cadence for the deployed tag. | Platform / release engineering | Locate or add a `build-image` + push workflow and pin the registry/tag. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Where and how is the deployed image built and published, and on what cadence is it rebased for base-image CVEs? | Determines the real supply-chain/patch posture, which buildpacks enable but the repo does not evidence. | Add a CI `build-image`/push job and document the rebase cadence. | Platform / release owner |
| Q2 | Should the image build run under Maven or Gradle as the canonical path? | Both build systems can produce the image; the canonical one is undeclared. | Resolve alongside [dual-build-system-maven-gradle](dual-build-system-maven-gradle.md). | Platform build owner |
