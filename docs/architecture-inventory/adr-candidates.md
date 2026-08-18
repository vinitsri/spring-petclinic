# ADR Candidates Backlog
**Total candidates: 15**

- **Project Key:** VE · **Project Name:** Petclinic platform
- **Stage:** Architecture discovery — ADR candidate identification
- **Branch:** `arch-discovery-57f5f6e8-e25d-475f-8aeb-42d3fc5622b6`
- **Date:** 2026-08-18
- **Scope:** One in-scope repository — `spring-petclinic` (a single-deployable Spring Boot 4.1.0
  monolith, Kubernetes/Maven name `petclinic`). Candidates below are **retroactive** ADRs that
  document architecturally significant decisions **already evidenced in the source tree**; there
  are no ADRs or approval records in the repository today (`docs/adr/` and `docs/decisions/` are
  absent — architecture-baseline §7), and no prior `adr-candidates.md` existed to adopt.

> **Adopt-existing note.** A first-pass search found **no existing ADRs** (`docs/adr/` absent) and
> **no prior candidates backlog** in this clone, so nothing is superseded — every entry below is a
> genuinely missing decision, not a re-proposal. Where a future ADR is authored, preserve the
> `ADR-` prefix and allocate ids sequentially.

> **Source-of-truth conflict (CAKE catalog vs code).** The CAKE catalog for working tenant
> `5K4DVCTX` returns a full ADR set (`ADR-001`…`ADR-012`: event-sourced CQRS + SNS outbox,
> PostgreSQL Row-Level Security multi-tenancy, Scheduling MFE via Webpack Module Federation,
> Twilio telehealth, waitlist, dual-path legacy/modern coexistence) for a **different platform** —
> the WebPT EMR appointment-scheduling modernization program. **None of it is realized in
> `spring-petclinic`** (verified live via `cake_graph_query`, 2026-08-18, and across
> `src/main/java/org/springframework/samples/petclinic/**`, `pom.xml`, `build.gradle`, `k8s/`).
> Per the mandatory Source-of-Truth rule, **code wins**: those catalog ADRs are **not** re-proposed
> here and are recorded as *Future/Intended State (Not Implemented)*. Whether that program is
> planned scope for this platform is unresolved (see ABQ Q1) and would generate a distinct second
> wave of candidates if confirmed.

## Candidate Summary Table
| ID | Title | Category | Impact | Dependencies | Priority |
|----|-------|----------|--------|--------------|----------|
| ADR-CANDIDATE-001 | Single-deployable layered monolith over distributed services | Orchestration | high | — | first-batch |
| ADR-CANDIDATE-002 | Server-rendered MVC (Thymeleaf) over client-side SPA/MFE | Integration | high | 001 | first-batch |
| ADR-CANDIDATE-003 | Aggregate-root persistence boundary (Owner aggregate) | Storage | high | 001 | first-batch |
| ADR-CANDIDATE-004 | Script-based schema provisioning without a migration tool | Storage | high | — | first-batch |
| ADR-CANDIDATE-005 | Multi-engine datastore via Spring profiles (H2/MySQL/PostgreSQL) | Storage | medium | 004 | second-batch |
| ADR-CANDIDATE-006 | In-process read-through cache over a networked cache | Storage | medium | 001 | second-batch |
| ADR-CANDIDATE-007 | No application-level authentication/authorization | Security | high | — | first-batch |
| ADR-CANDIDATE-008 | Management/diagnostics endpoint exposure posture | Security | medium | 007 | second-batch |
| ADR-CANDIDATE-009 | Cloud Native Buildpacks over a hand-written Dockerfile | Deployment | medium | — | second-batch |
| ADR-CANDIDATE-010 | Dual build system (Maven + Gradle) with undeclared canonical | Deployment | medium | — | second-batch |
| ADR-CANDIDATE-011 | Kubernetes deployment via plain manifests + service binding (no Helm) | Orchestration | medium | 001, 005 | second-batch |
| ADR-CANDIDATE-012 | First-class GraalVM native-image support | Deployment | low | 001 | backlog |
| ADR-CANDIDATE-013 | Build-time quality & supply-chain gates as governance | Other | medium | — | backlog |
| ADR-CANDIDATE-014 | Observability posture: Actuator-only, no metrics registry/tracing | Other | medium | 001 | backlog |
| ADR-CANDIDATE-015 | Layered (defense-in-depth) uniqueness enforcement | Storage | low | 003, 005 | backlog |

## Candidate Details

### ADR-CANDIDATE-001: Single-deployable layered monolith over distributed services
**Category:** Orchestration
**Impact:** high
**Why this is an ADR:** The platform is deliberately one Spring Boot process organised by Java
package (`owner`, `vet`, `model`, `system`) rather than a set of independently deployable
services. The alternatives were real and consequential — a service-per-domain decomposition, or a
modular monolith with enforced module boundaries — and the choice has lasting impact on
scalability, team topology, deployment cadence, and coupling (all domains share one JVM and one
schema). This is a foundational structural decision, not an implementation detail; nearly every
other candidate is scoped by it.
**Evidence:** `src/main/java/org/springframework/samples/petclinic/{owner,vet,model,system}/**`;
single `pom.xml`/`build.gradle` module (`settings.gradle` `rootProject.name = 'petclinic'`);
`k8s/petclinic.yml` (one `Deployment`/`Service`); architecture-baseline §1–§2; repo-inventory
rows 2, 4. No message broker, service client, or second deployable exists anywhere in the tree.
**Dependencies:** none (foundational).
**Recommended priority:** first-batch

### ADR-CANDIDATE-002: Server-rendered MVC (Thymeleaf) over client-side SPA/MFE
**Category:** Integration
**Impact:** high
**Why this is an ADR:** The UI is fully server-rendered (Spring MVC controllers returning Thymeleaf
views) with **no** `package.json`, JS framework, or micro-frontend. The realistic alternatives — a
React/Angular/Vue SPA over a JSON API, or the MFE/Module-Federation approach the CAKE catalog
describes for the other platform — carry very different consequences for delivery model, SEO,
client state, testing, and team skills. Choosing server-side MPA is an architecturally significant,
durable decision about the entire presentation tier.
**Evidence:** controllers under `owner/`, `vet/`, `system/` returning view names;
`src/main/resources/templates/**`; WebJars (Bootstrap 5.3.8, font-awesome 4.7.0) with SCSS→
`petclinic.css` via libsass; absence of `package.json`/npm build (architecture-baseline §4;
patterns `ui/server-rendered-mvc-views.md`).
**Dependencies:** ADR-CANDIDATE-001 (monolith bounds the UI-delivery choice).
**Recommended priority:** first-batch

### ADR-CANDIDATE-003: Aggregate-root persistence boundary (Owner aggregate)
**Category:** Storage
**Impact:** high
**Why this is an ADR:** Pets and visits are persisted **only** through the `Owner` aggregate root's
repository (`OwnerRepository.save`/`saveAndFlush`) rather than through independent `Pet`/`Visit`
repositories. This is a deliberate DDD consistency-boundary decision — the alternative (a repository
per entity, exposing pets/visits as independent aggregates) trades a cleaner CRUD surface for a
weaker consistency boundary. It has lasting impact on transaction scope, API shape, and how future
domains are modelled.
**Evidence:** `src/main/java/.../owner/Owner.java` (`@OneToMany` pets, cascade), `OwnerController`/
`PetController`/`VisitController` all routing writes through `OwnerRepository`;
architecture-baseline §1, §3; patterns `data/aggregate-root-persistence.md`.
**Dependencies:** ADR-CANDIDATE-001.
**Recommended priority:** first-batch

### ADR-CANDIDATE-004: Script-based schema provisioning without a migration tool
**Category:** Storage
**Impact:** high
**Why this is an ADR:** Schema and seed data are applied from checked-in per-engine SQL
(`spring.sql.init` running `db/{h2,mysql,postgres}/schema.sql` + `data.sql`) with
`spring.jpa.hibernate.ddl-auto=none`. The clear alternatives — Flyway or Liquibase versioned
migrations, or Hibernate auto-DDL — were **not** adopted. The trade-off is explicit: simplicity and
no extra dependency versus no migration history, no rollback, and a real drift risk across three
hand-maintained SQL variants. This is a durable data-lifecycle decision.
**Evidence:** `application.properties` (`spring.jpa.hibernate.ddl-auto=none`, `spring.sql.init`);
`src/main/resources/db/{h2,mysql,postgres}/{schema,data}.sql`; absence of Flyway/Liquibase in
`pom.xml`/`build.gradle`; architecture-baseline §3; patterns `data/script-based-schema-provisioning.md`.
**Dependencies:** none.
**Recommended priority:** first-batch

### ADR-CANDIDATE-005: Multi-engine datastore via Spring profiles (H2/MySQL/PostgreSQL)
**Category:** Storage
**Impact:** medium
**Why this is an ADR:** One artifact is designed to run against three relational engines (H2
in-memory by default, MySQL, PostgreSQL) selected by Spring profile at deploy time. The alternative
— committing to a single engine — would simplify the schema and let the app use engine-specific
features freely. Supporting three engines is a deliberate portability decision whose lasting cost is
maintaining three `schema.sql` variants and forgoing engine-specific optimisations (e.g. the
PostgreSQL functional unique index has no exact H2 equivalent).
**Evidence:** `application.properties` (`database=h2`), `application-mysql.properties`,
`application-postgres.properties`; `k8s/petclinic.yml` (`SPRING_PROFILES_ACTIVE=postgres`);
`docker-compose.yml` (`mysql:9.7`, `postgres:18.4`); three `db/*/schema.sql`; patterns
`deployment/profile-based-environment-configuration.md`.
**Dependencies:** ADR-CANDIDATE-004 (provisioning mechanism enables per-engine scripts).
**Recommended priority:** second-batch

### ADR-CANDIDATE-006: In-process read-through cache over a networked cache
**Category:** Storage
**Impact:** medium
**Why this is an ADR:** The read-mostly vet listing is fronted by a declarative `@Cacheable("vets")`
Caffeine/JCache cache local to the JVM. The alternative — a networked cache (Redis/Memcached) or no
cache — was not taken. The choice trades cross-instance cache coherency and shared eviction for
zero operational overhead, and it constrains horizontal scaling (each replica holds its own copy).
This is an architecturally significant caching-topology decision with lasting scaling implications.
**Evidence:** `src/main/java/.../vet/VetRepository.java:45,55` (`@Cacheable("vets")`);
`src/main/java/.../system/CacheConfiguration.java` (`@EnableCaching`, JCache/Caffeine, JMX stats);
Caffeine dependency in `pom.xml`; absence of Redis/Memcached; patterns
`data/read-through-in-process-cache.md`.
**Dependencies:** ADR-CANDIDATE-001.
**Recommended priority:** second-batch

### ADR-CANDIDATE-007: No application-level authentication/authorization
**Category:** Security
**Impact:** high
**Why this is an ADR:** The platform ships with **no** Spring Security (or equivalent) — no login,
no session auth, no role/permission checks; every endpoint is reachable without credentials. The
alternatives (Spring Security form/basic auth, OAuth2/OIDC, or an external gateway enforcing authn)
were not adopted. This is a deliberate, high-impact posture decision: it shapes the trust boundary,
determines whether the app can be internet-facing, and dictates what a production hardening effort
must add. It is a decision (with a documented rationale — a sample/demo app), not a mere absence.
**Evidence:** absence of `spring-boot-starter-security` in `pom.xml`/`build.gradle`; no filter chain
or security config in `system/`; architecture-baseline §5; repo-inventory row 10.
**Dependencies:** none.
**Recommended priority:** first-batch

### ADR-CANDIDATE-008: Management/diagnostics endpoint exposure posture
**Category:** Security
**Impact:** medium
**Why this is an ADR:** Actuator is fully exposed (`management.endpoints.web.exposure.include=*`)
and the H2 console is enabled, flagged in-config as dev/test conveniences. Whether these are
network-restricted in production is not evidenced in-repo. The decision — expose everything for
developer ergonomics vs. curate a minimal endpoint set / disable the DB console — carries a real
security trade-off (information disclosure, unauthenticated management surface). It is distinct from
ADR-CANDIDATE-007 (which concerns app authn) and warrants its own record of the exposure policy.
**Evidence:** `application.properties` (`management.endpoints.web.exposure.include=*`, H2 console
enabled, inline dev-only comments); `system/CrashController.java` (`/oups`); `k8s/petclinic.yml`
(`/livez`, `/readyz`); architecture-baseline §5–§6.
**Dependencies:** ADR-CANDIDATE-007 (shares the trust-boundary context).
**Recommended priority:** second-batch

### ADR-CANDIDATE-009: Cloud Native Buildpacks over a hand-written Dockerfile
**Category:** Deployment
**Impact:** medium
**Why this is an ADR:** The image is built with Cloud Native Buildpacks (`spring-boot:build-image`);
there is intentionally **no** Dockerfile. The alternative — a maintained Dockerfile (full control of
base image, layers, CVE patching cadence) — was not chosen. Buildpacks trade fine-grained control
for reproducible, maintenance-light images. This is a durable build/packaging decision that affects
supply-chain posture and how the deploy image is patched.
**Evidence:** `README.md` (buildpacks, "There is no Dockerfile"); absence of any `Dockerfile`;
`k8s/petclinic.yml` referencing image `dsyer/petclinic`; architecture-baseline §6. (The
build/publish pipeline for that specific tag is not in-repo — see ABQ.)
**Dependencies:** none.
**Recommended priority:** second-batch

### ADR-CANDIDATE-010: Dual build system (Maven + Gradle) with undeclared canonical
**Category:** Deployment
**Impact:** medium
**Why this is an ADR:** The repository maintains **both** a Maven (`pom.xml`) and a Gradle
(`build.gradle`) build, each wired into its own CI workflow, with no statement of which is canonical
for release. Supporting two build systems is a decision with ongoing cost (keeping dependency sets,
plugins, and SBOM/native config in sync) and an unresolved trade-off (contributor choice vs. single
source of truth). An ADR should record why both exist and which governs release.
**Evidence:** `pom.xml`, `build.gradle`, `settings.gradle`; `.github/workflows/maven-build.yml`,
`gradle-build.yml`; repo-inventory row 13 (dual build authority); architecture-baseline §6.
**Dependencies:** none.
**Recommended priority:** second-batch

### ADR-CANDIDATE-011: Kubernetes deployment via plain manifests + service binding (no Helm)
**Category:** Orchestration
**Impact:** medium
**Why this is an ADR:** Deployment uses hand-written `k8s/*.yml` (`NodePort` Service, plain
`Deployment`, a `postgres` Deployment + `servicebinding.io/postgresql` Secret) applied with
`kubectl apply`; no Helm chart, Kustomize overlay, or GitOps tooling. The alternative packaging
approaches (Helm/Kustomize/Argo) offer templating, environment overlays, and release management that
plain manifests lack. Choosing raw manifests + service binding is a deliberate operational decision
affecting environment promotion and rollback.
**Evidence:** `k8s/petclinic.yml`, `k8s/db.yml`; `.github/workflows/deploy-and-test-cluster.yml`
(`kubectl apply -f k8s/`); absence of Helm/Kustomize; architecture-baseline §6; architecture-views §4.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-005 (deployable + engine selection shape the manifests).
**Recommended priority:** second-batch

### ADR-CANDIDATE-012: First-class GraalVM native-image support
**Category:** Deployment
**Impact:** low
**Why this is an ADR:** The build carries GraalVM native-image support (`native-maven-plugin`,
`PetClinicRuntimeHints.java` registering reflection/resource hints). Maintaining a native path
alongside the JVM path is a decision with a real trade-off — fast startup / low memory footprint vs.
longer builds, reflection-hint maintenance, and a second artifact to test. It has lasting impact on
build pipeline and library-compatibility constraints, warranting a (low-impact) record.
**Evidence:** `native-maven-plugin` in `pom.xml`; `src/main/java/.../PetClinicRuntimeHints.java`;
repo-inventory "Documentation vs tree" (native build support); architecture-baseline §6.
**Dependencies:** ADR-CANDIDATE-001.
**Recommended priority:** backlog

### ADR-CANDIDATE-013: Build-time quality & supply-chain gates as governance
**Category:** Other
**Impact:** medium
**Why this is an ADR:** In the absence of ADRs, governance is enforced through the build:
Checkstyle, spring-javaformat, `nohttp`, the Maven enforcer plugin, and CycloneDX SBOM generation
all run in CI and fail the build on violation. Treating these as the platform's governance
mechanism — rather than review conventions or a separate policy engine — is a deliberate decision
with lasting impact on contribution flow and supply-chain assurance. An ADR should record the gate
set and the fail-the-build policy.
**Evidence:** `pom.xml` / `.mvn/` (checkstyle, spring-javaformat, nohttp, enforcer, CycloneDX);
`.github/workflows/**`; architecture-baseline §7; patterns
`testing/build-time-quality-and-supply-chain-gates.md`.
**Dependencies:** none.
**Recommended priority:** backlog

### ADR-CANDIDATE-014: Observability posture — Actuator-only, no metrics registry/tracing
**Category:** Other
**Impact:** medium
**Why this is an ADR:** Observability is limited to Spring Boot Actuator (health/metrics/JMX) plus
JCache statistics; there is **no** Micrometer metrics registry (e.g. Prometheus) and **no**
distributed tracing (OpenTelemetry/Zipkin). The alternative — wiring a metrics backend and tracing —
was not taken. For a single-process app this is defensible, but it is a decision with lasting impact
on production operability and should be recorded (including the condition under which metrics/tracing
would be added).
**Evidence:** Actuator present, JCache stats enabled (`CacheConfiguration.java`,
`application.properties`); absence of Micrometer registry / OpenTelemetry deps in
`pom.xml`/`build.gradle`; architecture-baseline §6; repo-inventory row 11.
**Dependencies:** ADR-CANDIDATE-001.
**Recommended priority:** backlog

### ADR-CANDIDATE-015: Layered (defense-in-depth) uniqueness enforcement
**Category:** Storage
**Impact:** low
**Why this is an ADR:** The "one pet name per owner" invariant is enforced **both** by an application
pre-check/validator **and** by a database unique constraint (`unique_owner_pet_name` on
`(owner_id, LOWER(name))` in PostgreSQL / `UNIQUE(owner_id, name)` in H2), with the app translating
the resulting `DataIntegrityViolationException` into a field error. The alternatives — enforce only
in the app (race-prone) or only in the DB (poor UX) — make this a genuine defense-in-depth decision.
Its lasting impact is a reusable convention for how invariants are enforced across engines; low
impact because it is narrow in scope.
**Evidence:** `src/main/java/.../owner/PetController.java` (`isDuplicatePetNameViolation`),
`PetValidator.java`; `db/postgres/schema.sql` (functional unique index), `db/h2/schema.sql`;
architecture-views §3.3; patterns `data/layered-uniqueness-enforcement.md`.
**Dependencies:** ADR-CANDIDATE-003 (aggregate write path), ADR-CANDIDATE-005 (per-engine constraint expression).
**Recommended priority:** backlog

## Recommended First Batch (3-5 ADRs)
Ordered by dependency (foundational first):
1. **ADR-CANDIDATE-001 — Single-deployable layered monolith over distributed services** (no deps; scopes all others)
2. **ADR-CANDIDATE-004 — Script-based schema provisioning without a migration tool** (no deps; high data-lifecycle impact)
3. **ADR-CANDIDATE-007 — No application-level authentication/authorization** (no deps; defines the trust boundary)
4. **ADR-CANDIDATE-002 — Server-rendered MVC (Thymeleaf) over client-side SPA/MFE** (depends on 001)
5. **ADR-CANDIDATE-003 — Aggregate-root persistence boundary (Owner aggregate)** (depends on 001)

## Generated (batch 1 of 3 — 2026-08-18)
The Recommended First Batch has been authored as ADR files under `docs/adr/` (descriptive
kebab-case stems, no numeric prefix per the platform naming rule):

- **ADR-CANDIDATE-001** → `docs/adr/single-deployable-layered-monolith.md` ✅ generated
- **ADR-CANDIDATE-004** → `docs/adr/script-based-schema-provisioning.md` ✅ generated
- **ADR-CANDIDATE-007** → `docs/adr/no-application-level-authentication.md` ✅ generated
- **ADR-CANDIDATE-002** → `docs/adr/server-rendered-mvc-thymeleaf-ui.md` ✅ generated
- **ADR-CANDIDATE-003** → `docs/adr/aggregate-root-persistence-boundary.md` ✅ generated

## Generated (batch 2 of 3 — 2026-08-18)
Five of the six second-batch candidates have been authored as ADR files under `docs/adr/`
(descriptive kebab-case stems, no numeric prefix). ADR-CANDIDATE-011 was deferred to batch 3 to keep
this batch at the ≤ 5-file limit:

- **ADR-CANDIDATE-005** → `docs/adr/multi-engine-datastore-spring-profiles.md` ✅ generated
- **ADR-CANDIDATE-006** → `docs/adr/in-process-read-through-cache.md` ✅ generated
- **ADR-CANDIDATE-008** → `docs/adr/management-diagnostics-endpoint-exposure.md` ✅ generated
- **ADR-CANDIDATE-009** → `docs/adr/cloud-native-buildpacks-image-build.md` ✅ generated
- **ADR-CANDIDATE-010** → `docs/adr/dual-build-system-maven-gradle.md` ✅ generated

### Remaining backlog (next batch)
Author these in batch 3 (≤ 5 per batch), preserving candidate→file traceability:

- **Batch 3 (deferred second-batch + backlog priority):** ADR-CANDIDATE-011 (Kubernetes plain
  manifests + service binding, no Helm — deferred from batch 2), ADR-CANDIDATE-012 (GraalVM
  native-image support), ADR-CANDIDATE-013 (build-time quality & supply-chain gates as governance),
  ADR-CANDIDATE-014 (observability posture — Actuator-only), ADR-CANDIDATE-015 (layered uniqueness
  enforcement).

## Merge Recommendations
- **ADR-CANDIDATE-004 + ADR-CANDIDATE-005** — if the author prefers a single "Database portability &
  provisioning strategy" record, these can be merged: the multi-engine choice (005) and the
  migration-tool-free provisioning (004) are two facets of the same portability posture. Kept
  separate here because each has distinct alternatives and can be decided independently.
- **ADR-CANDIDATE-007 + ADR-CANDIDATE-008** — candidates for merge into one "Security posture (dev
  demo)" ADR: application-authn absence and management/console exposure share a trust-boundary
  rationale. Kept separate so the authn decision (007, high impact) is not diluted by the endpoint
  exposure decision (008, medium impact).
- No duplicate candidates were identified within this backlog. No overlap with existing ADRs exists
  because the repository contains none, and the CAKE `ADR-001…012` set belongs to a different
  platform (see the source-of-truth note) — it is **not** merged in.

## Excluded as Implementation Details
- **Post-Redirect-Get form handling with Bean Validation** — a standard Spring MVC coding pattern
  (validate → redirect on success / re-render on error); no architecturally significant alternative
  with lasting platform impact. Catalogued as a pattern, not an ADR.
- **Content-negotiated dual representation (HTML + one JSON `/vets` endpoint)** — a single
  `@ResponseBody` endpoint; framework-idiomatic, not a platform-wide API-style decision.
- **Data-binding field allowlist (mass-assignment protection via `setAllowedFields`)** — defensive
  coding within controllers; a good practice, not a decision with weighed alternatives.
- **Request-scoped i18n across 11 locales (`?lang=` + `SessionLocaleResolver`)** — a localization
  capability/pattern; the mechanism is the Spring-idiomatic default with no significant competing
  option to record.
- **Caffeine cache eviction/TTL values, connection settings, seed `data.sql` values, port `8080`,
  package/class naming** — configuration values and naming conventions; explicitly outside ADR scope.
- **CAKE `ADR-001…012` (event-sourced CQRS, PostgreSQL RLS multi-tenancy, Scheduling MFE Module
  Federation, Twilio telehealth, waitlist, dual-path coexistence)** — excluded as current-state
  decisions: they govern a different platform and have **no realization** in this code
  (Future/Intended State, Not Implemented). Would become candidates only if that program is
  confirmed in scope (ABQ Q1).

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | These candidates are **retroactive** ADRs documenting decisions implicit in the code; no ADRs or approval records exist in-repo to adopt or supersede. | Verified absence of `docs/adr/`/`docs/decisions/` and of any prior `adr-candidates.md` (architecture-baseline §7; directory listing). | If an out-of-repo decision/approval register exists, some candidates may already be decided and would be re-proposals. | Confirm with the platform architecture owner whether an external decision register governs this repo. |
| A2 | `spring-petclinic` is the sole in-scope repository and maps 1:1 to the `petclinic` deployable, so no cross-repo decisions were missed. | Only one clone exists in the workspace; `k8s/petclinic.yml` `metadata.name: petclinic`, Maven `<name>petclinic</name>`. | Missing repos would leave platform-level ADR candidates (integration/contracts) unidentified. | Confirm the request `repos` list and CAKE service set with the platform owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a decision owner/accountable team to each candidate before ADR authoring. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Does the CAKE scheduling/EMR modernization program (event sourcing, RLS multi-tenancy, MFE Module Federation, telehealth, waitlist, dual-path coexistence — `ADR-001…012`) represent planned future scope for this platform? | If in scope, it generates a large **second wave** of ADR candidates absent from this backlog; if not, those catalog ADRs remain out-of-scope noise. Directly affects backlog completeness (`Total candidates`). | Not implemented in code; treated as Future/Intended State (Not Implemented) and excluded. | Product / architecture owner |
| Q2 | Should retroactive ADRs be authored for a reference/sample application at all, or only forward-looking decisions? | Determines whether the first batch proceeds as documentation of the status quo or is deferred until a real change forces a decision. | Proceed with the first batch — the monolith/UI/persistence/security decisions are load-bearing for any downstream evolution. | Platform architecture owner |
| Q3 | Is Maven or Gradle the canonical build/release path (input to ADR-CANDIDATE-010)? | The candidate cannot record a chosen option until the canonical build is declared; SBOM/native automation depends on it. | — | Platform build owner |

