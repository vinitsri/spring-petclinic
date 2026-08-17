# API Inventory Baseline — Petclinic Platform (Evidence-Based)

- **Project Key:** VE
- **Project Name:** Petclinic platform
- **Stage:** Architecture discovery — API / contract inventory (observed current-state only)
- **Branch:** `arch-discovery-57f5f6e8-e25d-475f-8aeb-42d3fc5622b6`
- **Date:** 2026-08-17
- **Scope:** HTTP/RPC endpoints, MCP tool contracts, and asynchronous event/message contracts
  exposed or consumed by the single `petclinic` deployable in this workspace.

> **Confidence legend:** `high` = directly evidenced by source code, config, or an explicit
> schema/DTO · `medium` = evidenced but with an inference step · `low` = weak/naming-only signal ·
> `unknown` = insufficient evidence.
>
> **Contract labelling:** request/response shapes are marked **observed** when reconstructed from a
> concrete DTO/entity/serializer read statically, and **inferred** when derived only from
> controller parameter names or framework behaviour.

## Method and grounding

Evidence basis: direct reading of the `spring-petclinic` source tree
(`src/main/java/org/springframework/samples/petclinic/**`), `src/main/resources/application.properties`,
`pom.xml`, and the prior baselines `docs/architecture-inventory/repo-inventory.md`,
`docs/architecture-inventory/architecture-views.md`,
`docs/architecture-inventory/baselines/architecture-baseline.md`, and
`docs/architecture-inventory/baselines/capability-baseline.md`. Capability links (C1–C7) reference
the capability identifiers defined in the capability baseline.

The workspace contains **one** repository, `spring-petclinic` (git URL
`https://github.com/vinitsri/spring-petclinic`, base ref `feature/VE-216/aidlc`), a single-deployable
Spring Boot monolith (Kubernetes/Maven deployable name `petclinic`). It is a server-rendered Spring
**MVC + Thymeleaf** application. Consequently the observed HTTP surface is dominated by
**HTML-view / form-submission endpoints**, not machine-readable JSON/RPC APIs. Exactly **one**
endpoint returns a machine-readable representation (`GET /vets`, `@ResponseBody`); one further
endpoint (`GET /vets.html`) returns a paginated HTML view of the same data. All remaining
application endpoints render or redirect between Thymeleaf views.

**Discovery passes executed (per the mandatory Contract discovery method):**
- Routers/controllers: `@Controller` + `@GetMapping`/`@PostMapping`/`@RequestMapping` across
  `owner/`, `vet/`, and `system/` packages (6 controllers).
- Higher-confidence contract sources searched and **not found**: no `openapi.*` / `swagger.*`,
  no `*.postman_collection.json`, no `.http` / `.rest` files, no `.proto` files (verified by
  filesystem search excluding `target/`).
- Response-DTO reconstruction: the `GET /vets` body was reconstructed statically from the
  `Vets` → `Vet` → `Person` → `BaseEntity` and `Specialty`/`NamedEntity` classes and their
  getter/JAXB annotations, so it is graded **observed (high)** rather than "unknown".
- MCP tool registrations, `@tool`/`@mcp.tool`/FastMCP decorators: searched, **none found**.
- Async messaging (Kafka/RabbitMQ/JMS/SNS/SQS/EventBridge/Redis Pub-Sub, `@KafkaListener`,
  `@RabbitListener`, `ApplicationEvent` domain publishers): searched, **none found** (the only
  `ApplicationEvent` hit is a Testcontainers `ApplicationPreparedEvent` listener in
  `src/test/java/.../PostgresIntegrationTests.java`, not a domain event).

### Source-of-truth conflict — CAKE catalog vs observed code (MANDATORY surfacing)

The CAKE catalog for the working tenant (`5K4DVCTX`) returns a large, internally-consistent **API**
model for a **different platform** — a WebPT EMR appointment-scheduling modernization program. The
graph exposes API/contract nodes such as `POST /v1/appointments/book`, `/v1/appointments`,
`POST /v1/calendar/query`, `GET /v1/appointment-types`, `/v1/providers/availability`,
`POST /v1/waitlist/entries`, `POST /v1/reminders/schedule`, `POST /v1/virtual-visits`, a
`Configuration API`, `Scheduler Service API`, `Notification Send API`, and domain events
(`AppointmentCancelled v1`, `AppointmentNoShow v1`, `AppointmentRescheduled v1`) produced/consumed by
`Scheduler Service`, `Appointment Slot Service`, `Waitlist Service`, `Notification Service`, and a
`Scheduling MFE`.

- **What the catalog claims:** a versioned (`/v1/*`) REST + event-driven, multi-service scheduling
  platform with an OpenAPI-specified contract surface and asynchronous appointment domain events.
- **What the source code shows:** none of it exists in `spring-petclinic`. There is no `/v1/*` or
  `/api/*` path, no appointment/scheduling/waitlist/notification endpoint, no OpenAPI/Swagger spec,
  no gRPC `.proto`, no message broker, and no event publisher/consumer anywhere in the source tree.
  The observed HTTP surface is the owners/pets/vets/visits Spring MVC controller set documented below.
- **Resolution (code wins):** per the Source-of-Truth rule, source code is authoritative. The entire
  CAKE scheduling/EMR API and event model is recorded as **Future/Intended State (Not Implemented)**
  for this repository and is **not** entered as an observed API, MCP tool, or event below. Whether
  that program is planned scope for this platform is `[unknown]` — see ABQ.

CAKE is treated as background narrative context only. It did not add or remove any endpoint, tool, or
event in this inventory; every contract below is grounded in code/config.

---

## Part 1 — HTTP / RPC API inventory

No gRPC/`.proto`, OpenAPI/Swagger, Postman, or `.http`/`.rest` sources exist; every entry is derived
from Spring MVC controller mappings. **View endpoints** render/redirect Thymeleaf views (server-side
HTML) and do not expose a machine-readable JSON/XML contract; their "Response" column records the
resolved view name / redirect target (observed from the controller return value). The one
**data endpoint** (`GET /vets`) returns a serialized object graph and is documented with a
reconstructed body schema.

| # | Service / repo | Method + path | Purpose | Request | Response | Auth | Error contract | Related capability | Confidence | Evidence |
|---|----------------|---------------|---------|---------|----------|------|----------------|--------------------|------------|----------|
| E1 | `petclinic` | `GET /` | Landing/welcome page | none | HTML view `welcome` (**observed**) | none | Default Spring Boot error page (`/error`) — **observed** (no custom handler) | C5 | high | `system/WelcomeController.java:31-34` |
| E2 | `petclinic` | `GET /oups` | Deliberately throws to demo error handling | none | Throws `RuntimeException` → default error view `error`, HTTP 500 (**observed**) | none | Default `BasicErrorController` `/error`, status 500 | C5, C7 | high | `system/CrashController.java:33-37` |
| E3 | `petclinic` | `GET /owners/new` | Render new-owner form | none | HTML view `owners/createOrUpdateOwnerForm` (**observed**) | none | Default `/error` — **observed** | C1 | high | `owner/OwnerController.java:73-76` |
| E4 | `petclinic` | `POST /owners/new` | Create an owner | Form body (**observed** from `Owner`): `firstName` (String, `@NotBlank`, ≤30), `lastName` (String, `@NotBlank`, ≤30), `address` (String, `@NotBlank`), `city` (String, `@NotBlank`), `telephone` (String, `@NotBlank`, `\d{10}`). `id` is disallowed by the binder. | On success: `302` redirect to `/owners/{id}`; on validation error: re-renders `owners/createOrUpdateOwnerForm` (**observed**) | none | Bean-validation errors re-rendered in the form via `BindingResult` (no JSON error body) — **observed** | C1 | high | `owner/OwnerController.java:79-88`; `owner/Owner.java:51-62`; `model/Person.java:31-39` |
| E5 | `petclinic` | `GET /owners/find` | Render owner search form | none | HTML view `owners/findOwners` (**observed**) | none | Default `/error` — **observed** | C1 | high | `owner/OwnerController.java:90-93` |
| E6 | `petclinic` | `GET /owners` | Paginated last-name owner search | Query params (**observed**): `page` (int, default `1`), `lastName` (String, bound via `Owner`, optional). Page size fixed at 5. | 0 results → re-render `owners/findOwners` with error; 1 result → `302` redirect `/owners/{id}`; many → HTML view `owners/ownersList` with `currentPage`/`totalPages`/`totalItems`/`listOwners` (**observed**) | none | `notFound` field error re-rendered in form — **observed** | C1 | high | `owner/OwnerController.java:95-121`; `owner/OwnerRepository.java` `findByLastNameStartingWith` |
| E7 | `petclinic` | `GET /owners/{ownerId}` | Show one owner (with pets & visits) | Path param (**observed**): `ownerId` (int) | HTML view `owners/ownerDetails` (`ModelAndView`) with `owner` (**observed**) | none | `IllegalArgumentException` if owner not found → default `/error`, HTTP 500 (no `@ResponseStatus`) — **observed** | C1, C2, C3 | high | `owner/OwnerController.java:154-162` |
| E8 | `petclinic` | `GET /owners/{ownerId}/edit` | Render edit-owner form | Path param (**observed**): `ownerId` (int) | HTML view `owners/createOrUpdateOwnerForm` (**observed**) | none | Default `/error`; missing owner → 500 — **observed** | C1 | high | `owner/OwnerController.java:123-126` |
| E9 | `petclinic` | `POST /owners/{ownerId}/edit` | Update an owner | Path param `ownerId` (int); form body same fields as E4 (**observed** from `Owner`) | On success: `302` redirect `/owners/{ownerId}`; on error or ID mismatch: re-render/redirect edit form (**observed**) | none | Validation + `id` `mismatch` field error re-rendered in form — **observed** | C1 | high | `owner/OwnerController.java:128-146` |
| E10 | `petclinic` | `GET /owners/{ownerId}/pets/new` | Render new-pet form | Path param (**observed**): `ownerId` (int); model attribute `types` = all `PetType` | HTML view `pets/createOrUpdatePetForm` (**observed**) | none | Missing owner → `IllegalArgumentException` → `/error` 500 — **observed** | C2 | high | `owner/PetController.java:104-109` |
| E11 | `petclinic` | `POST /owners/{ownerId}/pets/new` | Create a pet for an owner | Path param `ownerId` (int); form body (**observed** from `Pet`): `name` (String, `@NotBlank`), `birthDate` (LocalDate `yyyy-MM-dd`), `type` (`PetType` by id). `id` disallowed by binder. | On success: `302` redirect `/owners/{ownerId}`; on error: re-render `pets/createOrUpdatePetForm` (**observed**) | none | Duplicate-name (`duplicate`) and future-`birthDate` (`typeMismatch.birthDate`) field errors; `DataIntegrityViolationException` on `unique_owner_pet_name` re-rendered in form — **observed** | C2 | high | `owner/PetController.java:111-140`; `owner/Pet.java:48-54`; `model/NamedEntity.java:33-35` |
| E12 | `petclinic` | `GET /owners/{ownerId}/pets/{petId}/edit` | Render edit-pet form | Path params (**observed**): `ownerId` (int), `petId` (int) | HTML view `pets/createOrUpdatePetForm` (**observed**) | none | Missing owner/pet → `IllegalArgumentException` → 500 — **observed** | C2 | high | `owner/PetController.java:142-145` |
| E13 | `petclinic` | `POST /owners/{ownerId}/pets/{petId}/edit` | Update a pet | Path params `ownerId`, `petId` (int); form body same as E11 (**observed** from `Pet`) | On success: `302` redirect `/owners/{ownerId}`; on error: re-render `pets/createOrUpdatePetForm` (**observed**) | none | Same as E11 (duplicate-name, future-birthDate, unique-index violation) — **observed** | C2 | high | `owner/PetController.java:147-180` |
| E14 | `petclinic` | `GET /owners/{ownerId}/pets/{petId}/visits/new` | Render new-visit form | Path params (**observed**): `ownerId` (int), `petId` (int) | HTML view `pets/createOrUpdateVisitForm` (**observed**) | none | Missing owner/pet → `IllegalArgumentException` → 500 — **observed** | C3 | high | `owner/VisitController.java:88-91` |
| E15 | `petclinic` | `POST /owners/{ownerId}/pets/{petId}/visits/new` | Record a visit for a pet | Path params `ownerId`, `petId` (int); form body (**observed** from `Visit`): `date` (LocalDate `yyyy-MM-dd`, must be future), `description` (String, `@NotBlank`). `id` disallowed by binder. | On success: `302` redirect `/owners/{ownerId}`; on error: re-render `pets/createOrUpdateVisitForm` (**observed**) | none | Non-future `date` → `typeMismatch.visitDate` field error re-rendered in form — **observed** | C3 | high | `owner/VisitController.java:93-114`; `owner/Visit.java:38-43` |
| E16 | `petclinic` | `GET /vets.html` | Paginated veterinarian list (HTML) | Query param (**observed**): `page` (int, default `1`). Page size fixed at 5. | HTML view `vets/vetList` with `currentPage`/`totalPages`/`totalItems`/`listVets` (**observed**) | none | Default `/error` — **observed** | C4, C5 | high | `vet/VetController.java:47-50` |
| E17 | `petclinic` | `GET /vets` | **Data endpoint** — full veterinarian list as serialized object graph | none | Body **observed** (reconstructed from `Vets`→`Vet`→`Person`/`BaseEntity` + `Specialty`/`NamedEntity`): `{ "vetList": [ { "id": int, "firstName": string, "lastName": string, "specialties": [ { "id": int, "name": string } ], "nrOfSpecialties": int } ] }`. Serialized as JSON (Jackson) or XML (`@XmlRootElement` `Vets`/`Vet` via `MarshallingView`) by content negotiation. HTTP 200. | none | No custom handler; framework default. For `Accept: application/json`, Spring Boot's default error body `{timestamp,status,error,path}` applies — **observed** (default) | C4 | high | `vet/VetController.java:57-64`; `vet/Vets.java:33-45`; `vet/Vet.java:57-79`; `vet/Specialty.java`; `model/NamedEntity.java`; `model/BaseEntity.java:35-37` |
| E18 | `petclinic` | `GET /actuator`, `GET /actuator/{endpoint}` (e.g. `/actuator/health`, `/actuator/info`, `/actuator/metrics`, `/actuator/env`, `/actuator/beans`, `/actuator/mappings`) | Operational management/monitoring surface | Per-endpoint (framework-defined); not enumerated in application source | JSON per Actuator endpoint (framework-defined). All endpoints exposed via `management.endpoints.web.exposure.include=*` | none (no Spring Security dependency present) | Actuator default error responses — framework-defined | C7 | medium | `application.properties` `management.endpoints.web.exposure.include=*`; `pom.xml:44` `spring-boot-starter-actuator` |

**Auth note (all endpoints):** the project declares **no** `spring-boot-starter-security` dependency
and no security configuration; every endpoint above is served **unauthenticated**. This is recorded
as observed, not recommended — see ABQ.

**Error-contract note (all endpoints):** no `@ControllerAdvice`, `@ExceptionHandler`,
`@ResponseStatus`, or custom `ErrorController` exists in the source tree (verified by search).
Error handling is entirely Spring Boot's default: HTML clients receive the `error` view; JSON clients
receive the default `{timestamp,status,error,path}` body via `BasicErrorController`. Domain
`IllegalArgumentException`s (missing owner/pet) are unmapped and surface as HTTP 500.

---

## Part 2 — MCP Tool Contract Inventory

**No MCP tool contracts exist in this repository.**

The Contract discovery method was applied in full: searches for MCP tool registrations, tool
decorators (`@tool`, `@mcp.tool`, FastMCP), MCP server implementations, and tool wrapper/client
classes across the source tree returned **no matches**. There are no Python/Node MCP components; the
repository is a Java Spring Boot MVC monolith with no agent/tool-serving surface. This is a
**confirmed absence** (observed, high confidence), not an evidence gap.

The MCP-shaped entities that appear in the CAKE catalog for tenant `5K4DVCTX` belong to the unrelated
WebPT scheduling program and are recorded as **Future/Intended State (Not Implemented)** for this
repository (see the Source-of-truth conflict above). No MCP `exposes`/`consumes` edges are asserted.

---

## Part 3 — Event / message schema inventory

**No asynchronous event or message contracts exist in this repository.**

The Contract discovery method was applied in full: searches for message-broker clients and listeners
(Kafka/`@KafkaListener`, RabbitMQ/AMQP/`@RabbitListener`, JMS, AWS SNS/SQS, EventBridge, Redis
Pub/Sub), Spring `ApplicationEvent` domain publishers/`@EventListener` handlers, and topic/queue
definitions in config or deployment manifests returned **no domain-messaging matches**. The only
`ApplicationEvent` reference is an `ApplicationPreparedEvent` listener used to wire Testcontainers in
`src/test/java/org/springframework/samples/petclinic/PostgresIntegrationTests.java:103` — a test
harness concern, not a runtime domain event. All inter-component communication in `petclinic` is
**synchronous, in-process** (controller → Spring Data JPA repository → relational DB). This is a
**confirmed absence** (observed, high confidence), not an evidence gap.

The `AppointmentCancelled v1` / `AppointmentNoShow v1` / `AppointmentRescheduled v1` events in the
CAKE catalog belong to the unrelated WebPT scheduling program and are recorded as **Future/Intended
State (Not Implemented)** for this repository. No event `publishes`/`consumes` edges are asserted.

---

## Contract statements

Standalone relationship statements for knowledge-graph extraction. Only the observed data endpoint
produces a machine-readable contract; there are no MCP tools and no events to state.

- `petclinic` exposes the `GET /vets` HTTP data endpoint returning the `Vets` object graph (`vetList[]` of `Vet` with nested `specialties[]`).
- `petclinic` exposes the `GET /vets.html` HTTP endpoint rendering the `vets/vetList` view.
- `petclinic` exposes the owner HTTP endpoints `GET /owners/new`, `POST /owners/new`, `GET /owners/find`, `GET /owners`, `GET /owners/{ownerId}`, `GET /owners/{ownerId}/edit`, and `POST /owners/{ownerId}/edit`.
- `petclinic` exposes the pet HTTP endpoints `GET /owners/{ownerId}/pets/new`, `POST /owners/{ownerId}/pets/new`, `GET /owners/{ownerId}/pets/{petId}/edit`, and `POST /owners/{ownerId}/pets/{petId}/edit`.
- `petclinic` exposes the visit HTTP endpoints `GET /owners/{ownerId}/pets/{petId}/visits/new` and `POST /owners/{ownerId}/pets/{petId}/visits/new`.
- `petclinic` exposes the operational HTTP surface `GET /actuator/**` via `management.endpoints.web.exposure.include=*`.
- `petclinic` exposes no MCP tools.
- `petclinic` publishes no asynchronous events and consumes no asynchronous events.

---

## Gaps

| # | Item | Nature of gap | Status |
|---|------|---------------|--------|
| G1 | Consumers of `GET /vets` and `GET /vets.html` | No external client, integration test HTTP fixture, or OpenAPI consumer reference exists in-repo to identify who calls the data endpoint. | Producer observed; consumers **unknown — no in-repo evidence** |
| G2 | Actuator endpoint enumeration & per-endpoint contract (E18) | Endpoints are framework-provided via `include=*`; the concrete exposed set and payloads are not declared in application source. | **Framework-defined; not enumerated in source** |
| G3 | Production auth boundary | No `spring-boot-starter-security` and no security config in-repo; whether an external gateway/ingress enforces auth in production is not observable from this clone. | **Unverifiable — missing source evidence** (network/deploy layer) |
| G4 | CAKE scheduling/EMR API + event model | The working-tenant catalog describes a `/v1/*` REST + event-driven platform absent from the code. | **Future/Intended State (Not Implemented)** for this repo |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The `GET /vets` JSON body serializes exactly the getters read statically (`vetList[]` → `id`, `firstName`, `lastName`, `specialties[]{id,name}`, `nrOfSpecialties`). | Reconstructed from `Vets`/`Vet`/`Person`/`BaseEntity`/`Specialty` getters and default Jackson bean serialization; no test fixture pins the wire shape. | Downstream consumers/contract tests could bind the wrong field names or miss `nrOfSpecialties`/XML variant. | Add a `MockMvc` JSON assertion or capture a live `GET /vets` response. |
| A2 | Actuator exposes the full default endpoint set at `/actuator/**` because `include=*` is set. | `application.properties` `management.endpoints.web.exposure.include=*` with actuator on the classpath. | E18 over- or under-states the exposed operational surface. | Hit `/actuator` at runtime and read the discovery links. |
| A3 | All endpoints are unauthenticated as deployed. | No Spring Security dependency or config in the source tree. | If an external gateway enforces auth, the "none" auth column is wrong for the deployed system. | Confirm ingress/gateway auth with platform/security owner. |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Who consumes the `GET /vets` data endpoint (any external/UI client)? | Determines whether `GET /vets` is a real integration contract needing versioning/compatibility guarantees. | Unknown — no in-repo consumer found (G1). | Platform / API owner |
| Q2 | Does the CAKE scheduling/EMR API + event model (`/v1/*`, appointment domain events, MCP-style tools) represent planned future scope for this platform, or an unrelated catalog for a different product? | Determines whether a large set of catalog APIs/events are future scope vs out-of-scope noise for this repo. | Not implemented in code; recorded as Future/Intended State (Not Implemented). | Product / architecture owner |
| Q3 | Are `/actuator/**` endpoints and any error stack traces network-restricted in production? | `include=*` with no auth is a production exposure risk affecting the E18/C7 posture. | Assumed dev-only per inline config comments; production controls unverified. | Security / platform owner |
