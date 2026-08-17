# Capability Baseline — Petclinic Platform (Evidence-Based)

- **Project Key:** VE
- **Project Name:** Petclinic platform
- **Stage:** Architecture discovery — capability baseline (observed current-state only)
- **Branch:** `arch-discovery-57f5f6e8-e25d-475f-8aeb-42d3fc5622b6`
- **Date:** 2026-08-17
- **Authoritative scope:** This document is the **single authoritative capability reference** for this
  platform. It consolidates the capability list **and** the service→capability ownership mapping —
  there is no separate capability list or separate service-capability mapping document.

> **Confidence legend:** `high` = directly evidenced by runtime manifests, source code, or config ·
> `medium` = evidenced but with an inference step · `low` = weak/naming-only signal · `unknown` =
> insufficient evidence.
>
> **Evidence-order note.** Claims are graded strongest-first: (1) runtime/deployment manifests,
> (2) source code and imports, (3) configuration, (4) ADRs/docs, (5) naming/weak inference (marked
> as inference). Observed facts, inferred observations, assumptions, and unknowns are kept distinct.

## Method and grounding

Evidence basis: `docs/architecture-inventory/repo-inventory.md`,
`docs/architecture-inventory/architecture-views.md`, the per-repo summary
`docs/architecture-inventory/repo-summary/spring-petclinic.md`, and direct reading of the
`spring-petclinic` source tree. `docs/architecture-inventory/baselines/service-summaries.md` is
**not observed** on this branch; service ownership below is therefore derived from the repo
inventory, the repo summary, and code rather than from a service-summaries artifact.

The workspace contains **one** repository, `spring-petclinic` (git URL
`https://github.com/vinitsri/spring-petclinic`, base ref `feature/VE-216/aidlc`), a single-deployable
Spring Boot 4.1.0 monolith (Kubernetes/Maven deployable name `petclinic`). Because there is exactly
one service, every capability below is realized inside that one deployable; "ownership" distinguishes
Java **package/module** boundaries within the JVM, not separate services.

### Source-of-truth conflict — CAKE catalog vs observed code (MANDATORY surfacing)

The CAKE catalog for the working tenant (`5K4DVCTX`) returns a large, internally-consistent
capability model for a **different platform** — a WebPT EMR appointment-scheduling modernization
program. Catalog capability nodes include `Scheduling`, `Appointment Lifecycle Management`,
`Calendar Management`, `Provider Availability Management`, `Intelligent Slot Generation`,
`Waitlist Management`, `Appointment Reminders`, `Conversational Scheduling (Eva)`,
`BHC FHIR Partner Self-Scheduling`, `domain-event publishing` (`AppointmentScheduled` /
`AppointmentCancelled`), `HIPAA audit` / `Auditability`, `multi-tenant isolation`, and a
`Scheduling Capability Modernization` capability group referencing event-sourced Node.js services
and React Micro Frontends.

- **What the catalog claims:** an event-sourced, multi-tenant, MFE-based appointment-scheduling
  platform (Scheduler Service, Appointment Slot Service, Waitlist Service, Notification Service,
  Configuration API, Medplum/FHIR, Redis, Twilio Video, Idempotency-Key locking governed by
  `ADR-APPLICATION-059`, `ADR-006`).
- **What the source code shows:** none of it exists in `spring-petclinic`. There is no appointment
  or scheduling domain, no `/v1/*` or `/api/v1/*` API, no event bus / event store, no MFE or
  JavaScript build, no multi-tenant isolation, no auth/audit layer, and no Node.js service. The code
  is a server-rendered Spring MVC + Thymeleaf monolith over owners/pets/vets/visits (verified across
  `src/main/java/org/springframework/samples/petclinic/**`, `pom.xml`, `build.gradle`, `k8s/`).
- **Resolution (code wins):** per the Source-of-Truth rule, source code is authoritative. The entire
  CAKE scheduling/EMR capability model is recorded as **Future/Intended State (Not Implemented)** for
  this repository and is **not** entered as an owned capability in Sections 1–8. Whether that program
  is planned scope for this platform is `[unknown]` (a product/architecture decision) — see ABQ.
- The **domain narrative** CAKE holds that *does* match the code (a veterinary-clinic management app:
  vets and specialties, pet owners, pets, visits, browse/search) is consistent with the observed
  capabilities below and is used only as corroborating background, never as the sole basis for a
  claim.

CAKE is treated as background narrative context only. It did not add or remove any capability in this
baseline; every capability below is grounded in code/config.

---

## Section 1 — Capability-First View

Six capabilities are evidenced from code — four business capabilities (C1–C4), one presentation
capability (C5), and two cross-cutting technical/operational capabilities (C6–C7). All are realized
inside the single `petclinic` deployable.

**Capability: C1 — Pet Owner Management**
- **Description:** Register, search/find, view, and edit pet **owners** (name, address, city, 10-digit
  telephone). Provides paginated last-name search and single-owner detail views, and is the aggregate
  root that transitively exposes an owner's pets and visits.
- **Purpose / business value:** Maintain the clinic's customer records — the veterinary-clinic domain
  narrative (managing pet owners) is documented; concrete business-value language beyond that is
  **not observed**.
- **Confidence:** high
- **Evidence:** `src/main/java/.../owner/Owner.java`, `owner/OwnerController.java`
  (`/owners/new`, `/owners/find`, `/owners`, `/owners/{ownerId}`, `/owners/{ownerId}/edit`),
  `owner/OwnerRepository.java` (`findByLastNameStartingWith`); templates `owners/*.html`.

**Capability: C2 — Pet & Pet-Type Management**
- **Description:** Add and edit **pets** belonging to an owner, each classified by a **pet type**, with
  validation (name/type/birth-date required) and a unique-pet-name-per-owner rule. Pet types are
  resolved from a reference table via a formatter.
- **Purpose / business value:** Track the animals under each owner for clinical records; documented
  domain narrative only, no explicit business-value statement — **not observed**.
- **Confidence:** high
- **Evidence:** `owner/Pet.java`, `owner/PetType.java`, `owner/PetController.java`
  (`@RequestMapping("/owners/{ownerId}")` → `/pets/new`, `/pets/{petId}/edit`),
  `owner/PetValidator.java`, `owner/PetTypeFormatter.java`, `owner/PetTypeRepository.java`;
  templates `pets/createOrUpdatePetForm.html`.

**Capability: C3 — Veterinary Visit Recording**
- **Description:** Record dated **visits** (with a description) against a specific pet, reachable
  through the owner→pet path.
- **Purpose / business value:** Capture the clinical-visit history for a pet; documented domain
  narrative only — **not observed** beyond that.
- **Confidence:** high
- **Evidence:** `owner/Visit.java`, `owner/VisitController.java`
  (`GET|POST /owners/{ownerId}/pets/{petId}/visits/new`); template
  `pets/createOrUpdateVisitForm.html`; schema table `visits` (`db/*/schema.sql`).

**Capability: C4 — Veterinarian & Specialty Directory**
- **Description:** List veterinarians and their **specialties**, served both as a paginated HTML page
  (`/vets.html`) and as a JSON resource (`/vets`, `@ResponseBody`). Backed by an in-process cache.
- **Purpose / business value:** Expose the clinic's veterinary staff and specialties for browsing;
  documented domain narrative only — **not observed** beyond that.
- **Confidence:** high
- **Evidence:** `vet/Vet.java`, `vet/Specialty.java`, `vet/Vets.java`, `vet/VetController.java`
  (`/vets.html`, `/vets`), `vet/VetRepository.java:45,55` (`@Cacheable("vets")`); template
  `vets/vetList.html`; schema tables `vets`, `specialties`, `vet_specialties`.

**Capability: C5 — Clinic Web Presentation & Localization**
- **Description:** Server-rendered UI shell — welcome/home page, shared Thymeleaf layout fragments,
  Bootstrap/font-awesome styling (WebJars, SCSS→CSS), and request-scoped internationalization across
  11 locales via a `?lang=` switch.
- **Purpose / business value:** Deliver the human-facing clinic web UI; **not observed** as an
  explicit business statement.
- **Confidence:** high
- **Evidence:** `system/WelcomeController.java` (`GET /`), `system/WebConfiguration.java`
  (`SessionLocaleResolver`, `LocaleChangeInterceptor`), `src/main/resources/messages/messages_*.properties`
  (11 locales), `templates/fragments/layout.html`, `templates/welcome.html`, `src/main/scss/*`.

**Capability: C6 — Relational Persistence & Schema Provisioning (technical)**
- **Description:** Single relational schema accessed via Spring Data JPA / Hibernate, runnable on H2
  (default, in-memory), MySQL, or PostgreSQL selected by Spring profile. Schema and seed data are
  applied by `spring.sql.init` from per-engine `schema.sql`/`data.sql`; there is **no** migration tool
  (Flyway/Liquibase absent).
- **Purpose / business value:** Persist all domain data; a supporting technical responsibility, not a
  business capability — business purpose **not observed**.
- **Confidence:** high
- **Evidence:** `application.properties`, `application-mysql.properties`, `application-postgres.properties`;
  `src/main/resources/db/{h2,mysql,postgres}/schema.sql` + `data.sql`; JPA repositories under
  `owner/` and `vet/`; `pom.xml`/`build.gradle` (h2, mysql-connector-j, postgresql).

**Capability: C7 — Platform Operations & Observability (technical)**
- **Description:** Operational surface of the deployable — Spring Boot Actuator (all endpoints
  exposed), Kubernetes liveness/readiness probes (`/livez`, `/readyz`), the `vets` Caffeine/JCache
  cache (with JMX statistics), the H2 dev console, and a deliberate error-demo endpoint (`/oups`).
- **Purpose / business value:** Health, monitoring, caching, and diagnostics for operating the app;
  technical responsibility — business purpose **not observed**.
- **Confidence:** high (component presence); **medium** on production posture (see gaps)
- **Evidence:** `application.properties` (`management.endpoints.web.exposure.include=*`),
  `system/CacheConfiguration.java` (`@EnableCaching`, JCache/Caffeine `vets`),
  `system/CrashController.java` (`GET /oups`), `k8s/petclinic.yml` (`/livez`, `/readyz`);
  `pom.xml` (`spring-boot-starter-actuator`, Caffeine). No Micrometer registry / tracing dependency.

---

## Section 2 — Service Ownership Mapping

Single service (`petclinic`, repo `spring-petclinic`). "Owner" below denotes the Java **package**
that realizes the capability inside that one deployable; there are no secondary *services* — only
secondary *package* contributors, or shared cross-cutting concerns, are noted.

| Capability | Primary Owner (package within `petclinic` / repo `spring-petclinic`) | Secondary Contributors | Ownership Confidence | Evidence |
|------------|----------------------------------------------------------------------|------------------------|----------------------|----------|
| C1 Pet Owner Management | `owner` package (`OwnerController`, `Owner`, `OwnerRepository`) | `model` (shared `Person`/`BaseEntity` superclasses); `system` (i18n/layout) | high | `src/main/java/.../owner/`, `.../model/Person.java` |
| C2 Pet & Pet-Type Management | `owner` package (`PetController`, `Pet`, `PetType`, `PetValidator`, `PetTypeFormatter`, `PetTypeRepository`) | `owner` `OwnerRepository` (pets persisted via the owner aggregate) | high | `src/main/java/.../owner/Pet*.java`, `owner/PetController.java` |
| C3 Veterinary Visit Recording | `owner` package (`VisitController`, `Visit`) | `owner` `OwnerRepository` (visits saved through the owner aggregate) | high | `src/main/java/.../owner/Visit*.java` |
| C4 Veterinarian & Specialty Directory | `vet` package (`VetController`, `Vet`, `Specialty`, `Vets`, `VetRepository`) | `model` (shared superclasses); `system` (`CacheConfiguration` provides the `vets` cache) | high | `src/main/java/.../vet/`, `.../system/CacheConfiguration.java` |
| C5 Clinic Web Presentation & Localization | `system` package (`WelcomeController`, `WebConfiguration`) + `templates/` + `messages/` | all domain packages render into shared layout fragments | high | `src/main/java/.../system/WebConfiguration.java`, `templates/fragments/layout.html`, `messages/` |
| C6 Relational Persistence & Schema Provisioning | Cross-cutting — `spring.sql.init` + `db/*` + JPA repositories | `owner` and `vet` repositories; `model` base classes | high | `src/main/resources/db/*/schema.sql`, `application*.properties` |
| C7 Platform Operations & Observability | `system` package (`CacheConfiguration`, `CrashController`) + Actuator config + `k8s/` probes | Spring Boot Actuator (framework); k8s manifests | high | `application.properties`, `system/*.java`, `k8s/petclinic.yml` |

**Notes on shared or ambiguous ownership:**
- **`owner` package owns three business capabilities (C1–C3).** Owner, Pet, and Visit management all
  live in one package and persist through the **owner aggregate** (`OwnerRepository.saveAndFlush`),
  so their code/data ownership is genuinely shared rather than cleanly separated — the boundary
  between C1/C2/C3 is logical, not physical.
- **C6 and C7 are cross-cutting**, not owned by a single business package: persistence is exercised by
  every domain repository, and operations/observability is largely framework (Actuator) plus the
  `system` package.
- **No cross-service ownership exists** — a single deployable owns the single schema; the intra-schema
  foreign keys are intra-service relations, not cross-service coupling.
- **External/organizational ownership is `[unknown]`** — CAKE holds no repo→service or owning-team
  record for this clone (repo-inventory Gaps; ABQ B1).

---

## Section 3 — Requirement / Feature Traceability

Only observable features (controllers, routes, validators, config keys) are traced — this is not a
specification.

| Capability | Observable Feature / Requirement | Evidence Location | Confidence |
|------------|----------------------------------|-------------------|------------|
| C1 | Create owner (`GET/POST /owners/new`) | `owner/OwnerController.java:72,77` | high |
| C1 | Find/search owners by last name, paginated (`GET /owners/find`, `GET /owners`) | `owner/OwnerController.java:89,94`; `OwnerRepository.findByLastNameStartingWith` (`:136`) | high |
| C1 | View owner detail (`GET /owners/{ownerId}`) | `owner/OwnerController.java:169` | high |
| C1 | Edit owner with ID-mismatch guard (`GET/POST /owners/{ownerId}/edit`) | `owner/OwnerController.java:139,144-161` | high |
| C2 | Add pet (`GET/POST /owners/{ownerId}/pets/new`) with validation | `owner/PetController.java:100,107`; `owner/PetValidator.java` | high |
| C2 | Edit pet (`GET/POST /owners/{ownerId}/pets/{petId}/edit`) | `owner/PetController.java:139,144` | high |
| C2 | Pet-type resolution from reference data | `owner/PetTypeFormatter.java`, `owner/PetTypeRepository.java` | high |
| C3 | Record visit (`GET/POST /owners/{ownerId}/pets/{petId}/visits/new`) | `owner/VisitController.java:90,97` | high |
| C4 | Vet list HTML, paginated (`GET /vets.html`) | `vet/VetController.java:44` | high |
| C4 | Vet list JSON resource (`GET /vets`, `@ResponseBody`) | `vet/VetController.java:65-70` | high |
| C5 | Welcome/home page (`GET /`) | `system/WelcomeController.java:25` | high |
| C5 | Locale switch (`?lang=`) across 11 locales | `system/WebConfiguration.java`; `messages/messages_*.properties` | high |
| C6 | DB engine selection by Spring profile (`spring.profiles`, `database=`) | `application-mysql.properties`, `application-postgres.properties`, `application.properties` | high |
| C6 | Schema + seed provisioning via `spring.sql.init` | `db/{h2,mysql,postgres}/schema.sql`+`data.sql` | high |
| C7 | Actuator management endpoints (all exposed) | `application.properties` (`management.endpoints.web.exposure.include=*`) | high |
| C7 | K8s liveness/readiness probes (`/livez`, `/readyz`) | `k8s/petclinic.yml` | high |
| C7 | `vets` cache with JMX statistics | `system/CacheConfiguration.java`; `vet/VetRepository.java:45,55` | high |
| C7 | Error-demo endpoint (`GET /oups`) | `system/CrashController.java:31-34` | high |

---

## Section 4 — Code Evidence

- **Capability:** C1 Pet Owner Management
  - `owner/Owner.java` — JPA aggregate root; fields with Bean Validation (`@NotBlank`, telephone
    `@Pattern("\\d{10}")`); `@OneToMany` eager collection of pets (`owner_id` join).
  - `owner/OwnerController.java` — create/find/view/edit endpoints; disallows binding `id`
    (`setAllowedFields`); ID-mismatch guard on update.
  - `owner/OwnerRepository.java` — `findByLastNameStartingWith(...)` paginated query; `saveAndFlush`.

- **Capability:** C2 Pet & Pet-Type Management
  - `owner/Pet.java`, `owner/PetType.java` — pet entity and reference type.
  - `owner/PetController.java` — add/edit; catches `DataIntegrityViolationException` and re-checks
    `isDuplicatePetNameViolation` (`:128-132`, `:170-174`, `:202`).
  - `owner/PetValidator.java` — name/type/birth-date required (`rejectValue(..., REQUIRED)`).
  - `owner/PetTypeFormatter.java`, `owner/PetTypeRepository.java` — type lookup/formatting.

- **Capability:** C3 Veterinary Visit Recording
  - `owner/Visit.java` — visit entity (date + description).
  - `owner/VisitController.java` — new-visit form + submit under the owner/pet path; `owners.save(owner)`.

- **Capability:** C4 Veterinarian & Specialty Directory
  - `vet/Vet.java`, `vet/Specialty.java`, `vet/Vets.java` — vet entity, `@ManyToMany` specialties, JSON wrapper.
  - `vet/VetController.java` — `/vets.html` (paginated) and `/vets` (`@ResponseBody Vets`).
  - `vet/VetRepository.java` — `@Cacheable("vets")` on both `findAll` overloads (`:45,55`).

- **Capability:** C5 Clinic Web Presentation & Localization
  - `system/WelcomeController.java`, `system/WebConfiguration.java` (locale resolver + interceptor).
  - `templates/fragments/layout.html`, `templates/welcome.html`; `src/main/scss/*`;
    `src/main/resources/messages/messages_{de,en,es,fa,hi,ja,ko,pt,ru,tr}.properties` + default.

- **Capability:** C6 Relational Persistence & Schema Provisioning
  - `application.properties` (`ddl-auto=none`, `open-in-view=false`, batch fetch size),
    `application-mysql.properties`, `application-postgres.properties`.
  - `db/{h2,mysql,postgres}/schema.sql` + `data.sql`; tables `owners, pets, types, visits, vets,
    specialties, vet_specialties`.

- **Capability:** C7 Platform Operations & Observability
  - `system/CacheConfiguration.java` (`@EnableCaching`, JCache/Caffeine `vets`, statistics),
    `system/CrashController.java` (`/oups`).
  - `application.properties` (Actuator exposure), `k8s/petclinic.yml` (`/livez`, `/readyz`).

---

## Section 5 — Confidence Assessment

| Capability | Overall Confidence | Key Uncertainty | Evidence Basis |
|------------|--------------------|-----------------|----------------|
| C1 Pet Owner Management | high | None material | Source code (controllers, entity, repository) |
| C2 Pet & Pet-Type Management | high | None material | Source code + DB schema/constraint |
| C3 Veterinary Visit Recording | high | Visit field-level validation not fully enumerated here | Source code + schema `visits` |
| C4 Veterinarian & Specialty Directory | high | Cache TTL/eviction policy beyond defaults `[unknown]` | Source code + cache config |
| C5 Clinic Web Presentation & Localization | high | None material | Templates, SCSS, i18n bundles, web config |
| C6 Relational Persistence & Schema Provisioning | high | Runtime engine per environment (H2 vs Postgres); no migration tooling implies manual drift management | Config + `db/*` + manifests |
| C7 Platform Operations & Observability | medium | Production posture of open Actuator/H2; absence of metrics-registry/tracing | Config + `system` code + k8s probes |

**Overall capability inventory confidence: high.** Six code-evidenced capabilities in one deployable,
all corroborated by source, config, and (for C6/C7) runtime manifests. The single medium-confidence
axis is the **production operational posture** of C7, not the existence of the capability. No CAKE
scheduling/EMR capability contributed to this inventory (conflict resolved in favor of code).

---

## Section 6 — Gaps, Unknowns, and Assumptions

- **Gaps**
  - `[likely gap]` **Metrics export / distributed tracing** — Actuator is present but there is no
    Micrometer registry (e.g. Prometheus) and no tracing dependency (OpenTelemetry/Zipkin). Observability
    as a capability is partial. Evidence: absence in `pom.xml`/`build.gradle`.
  - `[likely gap]` **Schema-migration capability** — no Flyway/Liquibase; three hand-maintained
    `schema.sql` variants can drift. Evidence: `db/*` tree, no migration plugin.

- **Unknowns**
  - `[unknown]` **Runtime datastore per environment** — default is H2; `k8s/petclinic.yml` sets
    `SPRING_PROFILES_ACTIVE=postgres`. Which engine each environment actually runs is not fully
    evidenced in-repo.
  - `[unknown]` **Production restriction of Actuator (`*`) and the H2 console** — inline comments say
    dev/test-only; no in-repo evidence of production network restriction.
  - `[unknown]` **Repo→service / owning-team mapping** — CAKE holds none for this clone; organizational
    ownership of every capability is undetermined beyond code authorship.
  - `[unknown]` **Whether the CAKE scheduling/EMR program is planned scope** for this platform.

- **Assumptions**
  - `[assumption]` The `owner` package's three capabilities (C1–C3) are treated as distinct capabilities
    despite sharing one package and one aggregate; the split is inferred from controller/entity
    boundaries, not from a separately deployable unit.
  - `[assumption]` C6 and C7 are counted as platform capabilities (operational responsibilities) rather
    than mere framework plumbing, because persistence-engine selection and the open operational surface
    are architecturally material.

---

## Section 7 — Architectural Characteristics

Observed structural characteristics only (what *is*). No recommendations.

- **Capability:** C1 Pet Owner Management
  - **Coupling / isolation:** Tightly coupled to C2/C3 — Owner is the aggregate root; pets and visits
    are persisted through `OwnerRepository`. Shares the `model` superclasses (`Person`, `BaseEntity`)
    and the single relational schema (C6).
  - **Boundary clarity:** Module boundary is the `owner` Java package (clear); the API surface is a set
    of Spring MVC routes. Boundary against C2/C3 is **diffuse** (same package + same aggregate).
  - **Dependency surface:** Depends on C6 (persistence), C5 (presentation/i18n). No outbound service
    dependency.
  - **Observable quality signals:** Bean Validation on inputs; explicit ID-mismatch and binding-field
    guards; recent commit history touches owner search normalization (`bb37aad`). Test presence in
    `src/test/java/.../owner/` (inferred from standard layout).

- **Capability:** C2 Pet & Pet-Type Management
  - **Coupling / isolation:** Coupled to C1 (owner aggregate) and to reference data (`types`). Shares
    schema C6.
  - **Boundary clarity:** `owner` package; distinct controller/validator/entity classes, but not a
    separable module from C1.
  - **Dependency surface:** Depends on C1 (aggregate), C6 (persistence + the `unique_owner_pet_name`
    constraint), C5 (forms).
  - **Observable quality signals:** Dual-layer duplicate-name defense (app pre-check + DB unique index
    with `DataIntegrityViolationException` translation); dedicated `PetValidator`; recent fix commits
    (`88e37c1`).

- **Capability:** C3 Veterinary Visit Recording
  - **Coupling / isolation:** Coupled to C1/C2 through the owner→pet→visit path and the shared aggregate.
  - **Boundary clarity:** `owner` package; smallest of the domain capabilities; not independently
    deployable.
  - **Dependency surface:** Depends on C1/C2 (parent entities), C6 (persistence), C5 (form).
  - **Observable quality signals:** `@Valid` on the visit model; form re-render on binding errors.

- **Capability:** C4 Veterinarian & Specialty Directory
  - **Coupling / isolation:** Most isolated business capability — separate `vet` package, read-mostly,
    no write path in the app. Shares only `model` superclasses and schema C6; consumes the C7 `vets`
    cache.
  - **Boundary clarity:** Clear — own package, own repository, and a defined dual API surface (HTML +
    JSON `Vets`).
  - **Dependency surface:** Depends on C6 (persistence) and C7 (cache); depended on by C5 (renders vet
    list). No dependency from C1–C3.
  - **Observable quality signals:** Read-through caching via `@Cacheable("vets")`; JSON representation
    class (`Vets`) separate from the entity.

- **Capability:** C5 Clinic Web Presentation & Localization
  - **Coupling / isolation:** Cross-cutting — every domain capability renders through shared layout
    fragments and message bundles.
  - **Boundary clarity:** Diffuse by nature (presentation concern spread across `templates/`, `scss/`,
    `messages/`, and `system` config), but the i18n mechanism itself is localized in `WebConfiguration`.
  - **Dependency surface:** Depended on by C1–C4 (views). Depends on WebJars assets.
  - **Observable quality signals:** 11-locale coverage; no client-side JS framework (server-rendered);
    SCSS build discipline via `libsass`.

- **Capability:** C6 Relational Persistence & Schema Provisioning
  - **Coupling / isolation:** Shared by all domain capabilities — single schema, single connection
    configuration. This is the primary coupling surface across C1–C4.
  - **Boundary clarity:** Diffuse — persistence logic lives in each domain repository; schema lives in
    three engine-specific SQL variants with no single migration authority.
  - **Dependency surface:** Depended on by C1–C4. Depends on an external relational DB (JDBC).
  - **Observable quality signals:** `ddl-auto=none` + explicit `spring.sql.init`; per-engine schema
    variants (drift risk, no migration tool); `unique_owner_pet_name` enforced in DB.

- **Capability:** C7 Platform Operations & Observability
  - **Coupling / isolation:** Cross-cutting operational surface; the `vets` cache directly serves C4.
  - **Boundary clarity:** Partly framework (Actuator), partly `system` package, partly k8s manifests —
    diffuse across code and deployment config.
  - **Dependency surface:** Serves C4 (cache); exposes health/metrics for the whole deployable;
    consumed by k8s probes.
  - **Observable quality signals:** Actuator fully exposed (`*`) with dev-only inline comment; JCache
    statistics enabled; **no** metrics registry or tracing dependency; H2 console enabled.

---

## Section 8 — Behavioral Constraints

Only capabilities with clearly evidenced, externally observable behavioral rules appear. C5 (pure
presentation) and C7 (structural/operational only) have no domain behavioral constraints and are
omitted.

| Capability | Behavioral Constraint (standalone statement) | Type | Confidence | Evidence |
|------------|----------------------------------------------|------|------------|----------|
| C1 Pet Owner Management | Pet Owner Management must reject an owner whose first name, last name, address, or city is blank. | validation | high | `owner/Owner.java` (`@NotBlank` on the four fields) |
| C1 Pet Owner Management | Pet Owner Management must reject an owner whose telephone is not exactly 10 digits. | validation | high | `owner/Owner.java:61` (`@Pattern(regexp="\\d{10}")`) |
| C1 Pet Owner Management | Pet Owner Management must not apply an owner update when the submitted owner id does not match the path `ownerId`, and must return the user to the edit form. | validation | high | `owner/OwnerController.java:150-155` |
| C1 Pet Owner Management | Pet Owner Management must not accept a client-supplied `id` when binding an owner form. | validation | high | `owner/OwnerController.java:60` (`setAllowedFields`) |
| C2 Pet & Pet-Type Management | Pet & Pet-Type Management must reject a pet that has no name, no type, or no birth date. | validation | high | `owner/PetValidator.java:42,47,52` |
| C2 Pet & Pet-Type Management | Pet & Pet-Type Management must not allow two pets with the same name (case-insensitively) under the same owner, rejecting the duplicate as a `name` field error rather than persisting it. | invariant | high | `owner/PetController.java:112,128-132,170-174,202`; `db/postgres/schema.sql` unique index `unique_owner_pet_name` on `(owner_id, LOWER(name))`; `db/h2/schema.sql` `UNIQUE(owner_id, name)` |
| C3 Veterinary Visit Recording | Veterinary Visit Recording must validate a submitted visit and re-display the visit form (without saving) when validation fails. | validation | medium | `owner/VisitController.java:97-109` (`@Valid Visit`, save only on success) |
| C6 Relational Persistence & Schema Provisioning | Relational Persistence must not auto-generate or mutate schema at runtime (`ddl-auto=none`); schema and seed data are applied only from the configured `spring.sql.init` scripts. | invariant | high | `application.properties` (`spring.jpa.hibernate.ddl-auto=none`, `spring.sql.init`) |
| C6 Relational Persistence & Schema Provisioning | Relational Persistence must enforce the unique-pet-name-per-owner rule at the database level so it holds independently of the application check. | invariant | high | `db/postgres/schema.sql`, `db/h2/schema.sql` (`unique_owner_pet_name`) |

---

## Service Lookup Index

> Compact cross-reference for AI-agent and tooling consumption. For full capability descriptions and evidence, see Sections 1–4 above.

This platform is a **single deployable** (`petclinic`, repo `spring-petclinic`). There are no secondary *services* — the capabilities are owned by Java **packages** inside the one JVM (Section 2), so there is exactly one row.

| Service / Repo | Capabilities Owned (Primary) | Capabilities Supported (Secondary) | Confidence | Notes |
|----------------|------------------------------|-------------------------------------|------------|-------|
| `petclinic` | C1 Pet Owner Management, C2 Pet & Pet-Type Management, C3 Veterinary Visit Recording, C4 Veterinarian & Specialty Directory, C5 Clinic Web Presentation & Localization, C6 Relational Persistence & Schema Provisioning, C7 Platform Operations & Observability | — (no separate deployable exists) | medium | Single Spring Boot monolith (repo `spring-petclinic`, K8s/Maven deployable name `petclinic`); all seven capabilities realized in one JVM. Confidence held to **medium** by C7 (production posture of open Actuator/H2). Ownership is intra-JVM package-level, not cross-service; C1–C3 share the `owner` package + owner aggregate (diffuse boundary), and C6/C7 are cross-cutting. No CAKE repo→service / owning-team record (ABQ B1); CAKE holds only unrelated telehealth `Service` nodes and non-service `SystemComponent` nodes for PetClinic. |

### Ownership statements

`petclinic` implements the Pet Owner Management capability.
`petclinic` implements the Pet & Pet-Type Management capability.
`petclinic` implements the Veterinary Visit Recording capability.
`petclinic` implements the Veterinarian & Specialty Directory capability.
`petclinic` implements the Clinic Web Presentation & Localization capability.
`petclinic` implements the Relational Persistence & Schema Provisioning capability.
`petclinic` implements the Platform Operations & Observability capability.

Components in scope: petclinic

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Owner, Pet, and Visit are three distinct capabilities (C1–C3) despite sharing the `owner` package and the owner aggregate. | Distinct controllers, entities, routes, and validators; conventional PetClinic domain split. | Downstream impact analysis could over- or under-scope changes if these are actually one capability. | Confirm capability granularity with the platform architecture owner. |
| A2 | C6 (persistence/schema) and C7 (operations/observability) are platform capabilities, not mere framework plumbing. | Engine selection by profile and an open operational surface are architecturally material. | Baseline would over-count capabilities if these should be excluded. | Confirm capability-boundary policy with architecture owner. |
| A3 | The default runtime datastore is in-memory H2 unless a Spring profile overrides it (k8s uses `postgres`). | `application.properties` `database=h2`; `k8s/petclinic.yml` `SPRING_PROFILES_ACTIVE=postgres`. | C6 behavioral/schema reasoning could target the wrong engine (e.g. the Postgres functional unique index). | Confirm `SPRING_PROFILES_ACTIVE` per environment. |
| A4 | The open Actuator (`*`) and H2 console exposure is intended for development/testing only. | Inline comments in `application.properties`. | C7 posture would be a production security exposure. | Verify production config/ingress with platform/security owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record for this clone. | Organizational ownership fields for every capability (Section 2) and downstream graph ingestion. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Does the CAKE scheduling/EMR capability model (Scheduler Service, MFEs, appointment lifecycle, waitlist, HIPAA audit, event sourcing) represent planned future scope for this platform, or is it an unrelated catalog for a different product? | Determines whether a large set of catalog capabilities are future scope vs out-of-scope noise for this repo. | Not implemented in code; recorded as Future/Intended State (Not Implemented). | Product / architecture owner |
| Q2 | Are the fully-exposed Actuator endpoints and the H2 console network-restricted in production? | Affects the C7 operational-posture confidence and security boundary. | Assumed dev-only per inline comments; production controls unverified. | Security / platform owner |
| Q3 | Is schema-migration tooling (Flyway/Liquibase) expected for C6, given three hand-maintained per-engine `schema.sql` variants? | Manual schema drift risk across H2/MySQL/PostgreSQL. | None present today. | Platform / data owner |
| Q4 | Should metrics export (Micrometer/Prometheus) and/or distributed tracing be added to C7? | No registry/tracing dependency exists; affects observability completeness. | Not present; add only if observability requirements demand it. | Platform / observability owner |
