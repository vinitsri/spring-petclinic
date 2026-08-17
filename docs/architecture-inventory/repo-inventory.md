# Petclinic Platform — Architecture Inventory (Evidence-Based)

- **Project Key:** VE
- **Project Name:** Petclinic platform
- **Stage:** Architecture discovery — evidence-based inventory (no ADRs, no recommendations)
- **Branch:** `arch-discovery-57f5f6e8-e25d-475f-8aeb-42d3fc5622b6`
- **Date:** 2026-08-17

## Scope and method

This workspace contains **one** clone: `spring-petclinic` (git URL
`https://github.com/vinitsri/spring-petclinic`, base ref `feature/VE-216/aidlc`). It is
designated the writable artifact/ADR repository **and** it holds the platform's application
source code. Because the source it contains is the platform being inventoried, it is treated
as an **in-scope subject repository** here; the generated inventory files are written into it
only as the output location (`docs/architecture-inventory/`).

Each repo was analyzed with two passes — a **documentation pass** (`README.md`, `LICENSE.txt`,
in-tree DB setup notes) and a **repository pass** (manifests, source tree, deploy/CI clues) —
then reconciled. Source code is the source of truth; where the CAKE catalog or docs conflict
with code, the conflict is surfaced explicitly rather than reconciled silently.

**CAKE grounding.** The CAKE catalog for the working tenant confirms the domain narrative — a
veterinary clinic management application that manages veterinarians and their specialties,
registers and manages pet owners, tracks pets, and schedules/records veterinary visits, with
browse/search over clinic data. CAKE holds **no** repository→service mapping, owning-team, or
technology record for this specific `spring-petclinic` clone. A CAKE `Scheduler Service`
entity (appointment booking, tenant isolation, `/v1/...` APIs, MFE identity headers) surfaced
under the `visit` interest, but **no such service, API, or MFE exists in this repository's
code** — see *Documentation vs tree* and the per-repo Source-of-Truth note. It is recorded as
a non-applicable catalog artifact, not as a component of this source repo.

## Repository inventory table (14 dimensions, summary level)

One in-scope repository. `spring-petclinic` is a single-module Spring Boot monolith (not a
monorepo); the deployable name in Kubernetes is `petclinic`, and the Maven `<name>` is also
`petclinic`.

| # | Dimension | `spring-petclinic` (Repository: `spring-petclinic`, path: `.`) |
|---|-----------|----------------------------------------------------------------|
| 1 | Primary purpose | Sample veterinary-clinic management web app: owners, pets, vets/specialties, visits (`README.md`, `src/main/java/.../owner`, `.../vet`). |
| 2 | Main runtime/service type | Spring Boot 4.1.0 server-rendered web app (Spring MVC + Thymeleaf), Java 17; single deployable (`pom.xml`, `build.gradle`, `PetClinicApplication.java`). |
| 3 | Key entrypoints | `PetClinicApplication.main` (`@SpringBootApplication`); container HTTP port 8080 (`k8s/petclinic.yml`); test `main()` apps `PetClinicIntegrationTests`, `MysqlTestApplication`, `PostgresIntegrationTests`. |
| 4 | Important modules/packages | `owner`, `vet`, `model`, `system` under `org.springframework.samples.petclinic` (`src/main/java/...`). |
| 5 | External integrations | None to third-party services. Runtime deps: JDBC to H2/MySQL/PostgreSQL; WebJars (Bootstrap, font-awesome) (`pom.xml`, `build.gradle`). |
| 6 | Data stores / state | Relational DB via Spring Data JPA / Hibernate; H2 (default, in-memory), MySQL, PostgreSQL. **No migration tool** (Flyway/Liquibase absent); schema via `spring.sql.init` (`db/{h2,mysql,postgres}/schema.sql`+`data.sql`). Tables: `owners`, `pets`, `types`, `visits`, `vets`, `specialties`, `vet_specialties`. |
| 7 | Messaging / async / events | **None** — no SNS/SQS/Kafka/RabbitMQ/EventBridge deps or topics in code/config. |
| 8 | APIs exposed / consumed | Server-side HTML endpoints (`/owners*`, `/pets*`, `/vets.html`), one JSON endpoint `/vets` (`@ResponseBody`), diagnostics `/oups`, Actuator `/actuator/*` (all exposed), `/h2-console`, k8s probes `/livez` `/readyz`. Consumes none. |
| 9 | Deployment / runtime clues | `docker-compose.yml` (mysql, postgres); `k8s/petclinic.yml` + `k8s/db.yml` (image `dsyer/petclinic`, service binding, postgres); GH Actions `maven-build.yml`, `gradle-build.yml`, `deploy-and-test-cluster.yml`; buildpacks (no Dockerfile); `.devcontainer`, `.gitpod.yml`. |
| 10 | Security / auth | **No authentication/authorization** — no Spring Security dependency. Actuator fully exposed and H2 console enabled (README/`application.properties` flag these as dev-only). |
| 11 | Observability / logging / tracing | Spring Boot Actuator (health/metrics/JMX); JCache statistics enabled; Caffeine cache for `vets`; SLF4J/Logback default. No Micrometer registry, tracing, or Prometheus dependency. |
| 12 | Arch-decision files / feature flags | No ADR/`docs/` present. Governance via build tooling (checkstyle, spring-javaformat, nohttp, enforcer, CycloneDX SBOM). **No feature-flag system** (no LaunchDarkly/Unleash/Split/Flagsmith/custom flags). |
| 13 | Open questions / ambiguities | Dual Maven+Gradle build authority; deploy image ownership; production posture for open Actuator/H2 console — see ABQ. |
| 14 | Frontend stack | Server-side Thymeleaf templates + Bootstrap 5.3.8 & font-awesome 4.7.0 (WebJars); SCSS → `petclinic.css` via libsass. **No JS framework, no `package.json`, no MFE/federation.** |

## Cross-repo relationships

Only one repository is present, so there are no inter-repository call, shared-library, or
shared-datastore relationships to record. Notable **intra-repository** couplings:

- **Shared domain base** — `owner` and `vet` packages both extend `model.Person` /
  `model.BaseEntity` / `model.NamedEntity` (`src/main/java/.../model/`).
- **Shared data store** — all domains persist to one relational schema; the app runs against
  H2, MySQL, or PostgreSQL selected by Spring profile (`application-mysql.properties`,
  `application-postgres.properties`).
- **Intra-schema foreign keys** (single service, single schema — not cross-service coupling):
  `pets.owner_id → owners.id`, `pets.type_id → types.id`, `visits.pet_id → pets.id`,
  `vet_specialties.vet_id → vets.id`, `vet_specialties.specialty_id → specialties.id`
  (`src/main/resources/db/*/schema.sql`).

## Suspected platform subsystems

Grouping is by Java package boundary within the single deployable; these are logical modules,
not separately deployable services:

- **Clinic customer subsystem** (`owner` package) — `Owner`, `Pet`, `PetType`, `Visit`,
  `OwnerController`, `PetController`, `VisitController`, repositories, `PetValidator`,
  `PetTypeFormatter`.
- **Veterinarian subsystem** (`vet` package) — `Vet`, `Specialty`, `Vets`, `VetController`,
  `VetRepository`.
- **Platform/system subsystem** (`system` package) — `WelcomeController`, `CrashController`,
  `CacheConfiguration`, `WebConfiguration` (i18n).

## Gaps / unknowns

- **Deploy image provenance** — `k8s/petclinic.yml` references `dsyer/petclinic`; the repo has
  no Dockerfile and README relies on buildpacks, so the image build/publish pipeline for that
  tag is not observable in this clone. **Needs validation.**
- **Build-tool authority** — both Maven (`pom.xml`) and Gradle (`build.gradle`) are present and
  wired into CI; which is canonical for release is not stated. **Unknown.**
- **Production security posture** — Actuator (`management.endpoints.web.exposure.include=*`)
  and the H2 console are enabled; whether an external layer restricts them in production is not
  in this repo. **Needs validation.**
- **CAKE repo→service mapping** — the catalog does not map this repo to a service/component or
  record an owning team; ownership beyond code authorship is **Unknown**.

## Documentation vs tree (platform-level patterns)

- **README claim not reflected in tree (Stale doc):** README says "There is no `Dockerfile`" and
  points to buildpacks — consistent with the tree; however README does **not** mention the
  `k8s/` manifests, the `deploy-and-test-cluster.yml` workflow, `.devcontainer`, or the Gradle
  build, all of which exist on disk (disk-only components).
- **Disk-only components not in README:** `k8s/petclinic.yml`, `k8s/db.yml`, GraalVM native
  build support (`native-maven-plugin`, `PetClinicRuntimeHints.java`), CycloneDX SBOM plugin,
  and 11-locale i18n message bundles (`src/main/resources/messages/`).
- **Catalog vs code conflict:** the CAKE `Scheduler Service` (appointment booking, tenant
  isolation, `/v1/...` APIs, MFEs) has no realization in this repository — no such module,
  endpoint, message topic, or frontend exists. Labelled *Future/Intended State (Not
  Implemented)* / not applicable to this source clone.

## Coverage

Enumerated from workspace manifests (`pom.xml` single module `spring-petclinic`; `build.gradle`
+ `settings.gradle` `rootProject.name = 'petclinic'`; no `apps/*`, `services/*`, `libs/*`,
`packages/*`, `pnpm-workspace.yaml`, `nx.json`, or `go.work`).

| Repository | Projects enumerated | Documented | Excluded (reason) |
|-----------|--------------------|------------|-------------------|
| `spring-petclinic` | 1 (`spring-petclinic` / `petclinic` — single Spring Boot module) | 1 | 0 |

**Total:** 1 repository, 1 project enumerated, 1 documented, 0 excluded.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `spring-petclinic` is the sole in-scope platform repository. | Only one clone exists in the workspace; the `repos` field yielded a single entry. | Missing repositories would leave the platform inventory incomplete. | Confirm the request's `repos` list and CAKE tenant repository set with the platform owner. |
| A2 | The Kubernetes deployable is named `petclinic` and maps 1:1 to this repo. | `k8s/petclinic.yml` `metadata.name: petclinic`; Maven `<name>petclinic</name>`. | Entity naming in the knowledge graph would mis-link service to repo. | Confirm against the deployment registry / CAKE service node once one exists. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service/owning-team record for this clone. | Ownership and repo-classification fields in downstream graph ingestion. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Is Maven or Gradle the canonical build/release path? | Affects downstream build, SBOM, and native-image automation. | — | Platform build owner |
| Q2 | How is the `dsyer/petclinic` container image built and published? | The deploy manifest depends on it but no build pipeline is in-repo. | Likely buildpacks (`spring-boot:build-image`) per README, unverified. | Release/DevOps owner |
| Q3 | Are Actuator (`*` exposed) and the H2 console restricted in production? | Open management endpoints and DB console are security-sensitive. | Assumed dev-only per inline comments; production controls unverified. | Security / platform owner |
| Q4 | Does the CAKE `Scheduler Service` represent a planned future capability for this platform? | Determines whether appointment-booking is future scope vs unrelated catalog noise. | Not implemented in code; treat as future/intended or out-of-scope. | Product / architecture owner |
