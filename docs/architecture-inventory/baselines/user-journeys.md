# User Journeys Baseline — Petclinic Platform (Evidence-Based)

- **Project Key:** VE
- **Project Name:** Petclinic platform
- **Stage:** Architecture discovery — user-journey / navigation inventory (observed current-state only)
- **Branch:** `arch-discovery-57f5f6e8-e25d-475f-8aeb-42d3fc5622b6`
- **Date:** 2026-08-17
- **Scope:** How users move through the single `petclinic` deployable — server-side route sequences,
  template/redirect navigation edges, and the ordered navigation evidence captured in the repo's test
  assets — across the one repository in this workspace.

> **Confidence legend:** `high` = directly evidenced by source code, template, config, or a test ·
> `medium` = evidenced but with an inference step · `low` = weak/naming-only signal ·
> `unknown` = insufficient evidence.
>
> **Evidence-type labels:** **Confirmed** = the multi-step sequence is evidenced by a human-authored
> ordered test scenario (the JMeter plan) · **Partial** = some steps/edges are test-confirmed and
> others are inferred from router adjacency · **Inferred** = the sequence is derived only from router
> adjacency (template link / controller redirect), not from an ordered test.
>
> **Edge-strength note.** Individual redirect *edges* (e.g. `POST /owners/new` → `/owners/{id}`) are
> additionally verified by MockMvc controller tests that assert the redirect target. These confirm a
> single navigation *edge*, not a multi-page *sequence*; where noted as "test-confirmed edge" they
> raise confidence in that one transition.

## Method and grounding

Evidence basis: direct walk of the `spring-petclinic` source tree — Spring MVC controllers under
`src/main/java/org/springframework/samples/petclinic/{owner,vet,system}/**`, Thymeleaf templates
under `src/main/resources/templates/**`, and the test tree under `src/test/**` (JMeter plan, MockMvc
controller tests, RestTemplate integration tests) — cross-referenced against the three prior
baselines: `docs/architecture-inventory/baselines/ui-inventory.md` (routes/screens, rendering model),
`docs/architecture-inventory/baselines/capability-baseline.md` (capability ownership C1–C7), and
`docs/architecture-inventory/baselines/api-inventory.md` (endpoint identifiers E1–E18). Route
identifiers `E1`–`E18` and capability identifiers `C1`–`C7` below are those defined in those
baselines and are used as cross-references rather than re-derived.

The workspace contains **one** repository, `spring-petclinic` (git URL
`https://github.com/vinitsri/spring-petclinic`, base ref `feature/VE-216/aidlc`), a single-deployable
**server-rendered Spring MVC + Thymeleaf** monolith (Kubernetes/Maven deployable name `petclinic`).
Navigation is therefore **server-side**: full-page GET requests, HTML form POSTs, and controller
`redirect:` responses. There is **no** client-side router, SPA, or micro-frontend (established in the
UI inventory), so "route sequences" here means the ordered set of server routes a user traverses via
navbar links, in-page links, form submissions, and post-submit redirects.

### Step 0 — Prior context retrieved from CAKE (background narrative only)

The CAKE catalog for the working tenant (`5K4DVCTX`) was queried for documented user journeys,
business workflows, personas/actors, and cross-service choreography (a graph query over
`Process`/`Workflow`/`Persona`/`Actor`/`Journey` node types, and a `cake_search` for PetClinic
end-to-end journeys). Results:

- **No PetClinic-specific user-journey, persona, or ordered-workflow node was returned.** The graph's
  `Process` nodes belong to unrelated corpora (AUTOSAR Adaptive Platform methodology, EU
  cyber-resilience conformity-assessment regulation) and to the WebPT EMR scheduling program
  (`DO1 — Appointment Booking and Scheduling`, `Event-Driven Integration`, `async post-commit`,
  `Routing-mode dual-path`). The workflow drill-down returned the WebPT `Scheduling capability`
  (Appointment Lifecycle Management, Calendar management, React MFE) and `adlc-intent-planning`
  decision records — none of which exist in this repository.
- **`cake_search` explicitly returned no documented end-to-end journeys** for PetClinic; it offered
  only generic capability descriptions ("manage veterinarians and specialties", "register/manage pet
  owners", "track pets", "schedule and record veterinary visits", "browse and search clinic data").
  It also surfaced WebPT front-desk/scheduling personas ("front-desk/scheduling staff",
  "Add Appointment modal", "DO-1 Provider Selection UI Component") that are **not** implemented here.

Consistent with the API, capability, and UI baselines, the veterinary-clinic **domain narrative** CAKE
holds (owners, pets, visits, vets, browse/search) matches the code and is used only as corroborating
background. Every journey, route, hand-off, and shared-state claim below is grounded in code/tests.

### Source-of-truth conflict — CAKE catalog vs observed code (MANDATORY surfacing)

- **What the catalog implies:** documented appointment-scheduling journeys with defined personas
  (front-desk/scheduling staff, provider administrators), an `Add Appointment modal`, provider-
  selection UI flows, and event-choreographed cross-service booking workflows
  (`DO1 — Appointment Booking and Scheduling`, `AppointmentScheduled`/`AppointmentCancelled` events).
- **What the source code shows:** none of it exists in `spring-petclinic`. There is no appointment,
  scheduling, waitlist, or provider-availability route; no modal or client-side flow (server-rendered
  Thymeleaf only); and no event choreography (all navigation is synchronous, in-process controller →
  JPA repository → DB). The observed navigable surface is the owners/pets/vets/visits route set
  (E1–E18) documented in the API inventory.
- **Resolution (code wins):** per the Source-of-Truth rule, source code is authoritative. The CAKE
  scheduling personas and appointment journeys are recorded as **Future/Intended State (Not
  Implemented)** for this repository and are **not** entered as observed journeys, personas, or edges
  below. Whether that program is planned scope is `[unknown]` — see ABQ (mirrors the prior baselines).

## Step 1 — Navigation sources located (absence demonstrated, not assumed)

One repo (`spring-petclinic`) is in scope; it is both the artifact repo and the only code clone.
Every pattern in the task's search list was checked. The table records the exact glob/path inspected
and what was found — "none found" rows are the result of a filesystem walk of the whole repo
(excluding `.git` and build output), not a root-only glance.

| Navigation-source category | Path(s) / pattern(s) inspected | Result |
|----------------------------|--------------------------------|--------|
| JS router config (React/Vue/Angular/Next/Nuxt) | `src/router/**`, `src/routes.*`, `app/router.*`, `routes/index.*`; `<Route>`/`createBrowserRouter`/`createRouter`/`RouterModule.forRoot`; `pages/`, `app/` (App Router) | **none found** — no `package.json`, no JS framework anywhere (per ui-inventory) |
| Server-side route files (Rails/Laravel/Django/Express) | `routes.rb`, `routes/web.php`, `urls.py`, Express `app.get/post/use` | **none found** — not a Ruby/PHP/Python/Node app |
| **Server-side routing (Spring MVC)** | `@Controller` + `@GetMapping`/`@PostMapping`/`@RequestMapping` in `src/main/java/.../{owner,vet,system}/**` | **found** — 6 controllers own routes E1–E18 (see Step 2) |
| Navigation guards / auth middleware | `router/guards.*`, `middleware/auth.*`; `beforeEach`/`canActivate`/`requireAuth`/`redirectIfAuthenticated` | **none found** — no Spring Security dependency/config (api-inventory G3); the only request interceptor is `LocaleChangeInterceptor` (i18n, not a nav guard) in `system/WebConfiguration.java` |
| E2E suites (Cypress/Playwright/etc.) | `cypress/e2e/`, `cypress/integration/`, `tests/e2e/`, `e2e/`, `playwright/`, `__tests__/e2e/` | **none found** (directory search over whole repo) |
| E2E / feature test files | `*.cy.*`, `*.spec.js`, `*.spec.ts`, `*.e2e.*`, Gherkin `*.feature` | **none found** |
| **Ordered HTTP navigation scenario (load test)** | `src/test/jmeter/petclinic_test_plan.jmx` | **found** — a single ordered ThreadGroup of HTTP samplers walking Home→Vets→Find→Owner→Edit→Pet→Visit (the strongest sequenced-navigation evidence in the repo) |
| **Route-transition tests (MockMvc)** | `src/test/java/.../{owner,vet,system}/*ControllerTests.java` | **found** — assert view names and `redirect:`/`redirectedUrl` targets (confirm individual navigation edges) |
| **Full-stack HTTP tests (RestTemplate)** | `src/test/java/.../PetClinicIntegrationTests.java` | **found** — single-request GETs (`/owners/1`, `/owners?lastName=`), not multi-step sequences |
| Navigation-bearing templates | `<a th:href=...>`, `<form th:action=...>`, navbar links in `src/main/resources/templates/**` | **found** — navbar in `fragments/layout.html`; in-page links/forms in `owners/**`, `vets/**` (see Step 2) |
| Programmatic client navigation | `navigate(`, `router.push(`, `redirect(`, `<Link to=>`, `<router-link>` | **none found** — no client-side JS navigation (server `redirect:` is the only programmatic navigation, in controllers) |

**Summary of navigation evidence available:** (1) server-side route ownership in 6 Spring MVC
controllers; (2) template navigation edges (navbar + in-page links + form actions); (3) controller
`redirect:` edges; (4) one human-authored **ordered** navigation scenario (`petclinic_test_plan.jmx`);
(5) per-edge redirect assertions in MockMvc controller tests. No browser-driven e2e/Cypress/Playwright
or Gherkin scenario exists — this is a demonstrated absence, and the JMeter plan is the substitute
ordered-sequence evidence used to mark journeys **Confirmed**.

## Step 2 — Route sequences and adjacency

### Route inventory (route → component/controller owner → capability)

All routes are read directly from controller `@GetMapping`/`@PostMapping`/`@RequestMapping` values
and the resolved Thymeleaf view / `redirect:` return. Capability column cross-references
capability-baseline (C1–C7); route ids cross-reference api-inventory (E1–E18).

| Route (method) | View / redirect (evidence) | Controller owner | Capability | Route id |
|----------------|----------------------------|------------------|------------|----------|
| `GET /` | view `welcome` | `system/WelcomeController.java:25-27` | C5 | E1 |
| `GET /oups` | throws → default `error` view (HTTP 500) | `system/CrashController.java:31` | C5, C7 | E2 |
| `GET /owners/find` | view `owners/findOwners` | `owner/OwnerController.java:89-91` | C1 | E5 |
| `GET /owners` | 0 → re-render `findOwners`; 1 → `redirect:/owners/{id}`; many → view `owners/ownersList` | `owner/OwnerController.java:94-130` | C1 | E6 |
| `GET /owners/new` | view `owners/createOrUpdateOwnerForm` | `owner/OwnerController.java:72` | C1 | E3 |
| `POST /owners/new` | `redirect:/owners/{id}` (or re-render form) | `owner/OwnerController.java:77-86` | C1 | E4 |
| `GET /owners/{ownerId}` | view `owners/ownerDetails` | `owner/OwnerController.java:169` | C1 (surfaces C2, C3) | E7 |
| `GET /owners/{ownerId}/edit` | view `owners/createOrUpdateOwnerForm` | `owner/OwnerController.java:139` | C1 | E8 |
| `POST /owners/{ownerId}/edit` | `redirect:/owners/{ownerId}`; on id-mismatch `redirect:/owners/{ownerId}/edit` | `owner/OwnerController.java:144-161` | C1 | E9 |
| `GET /owners/{ownerId}/pets/new` | view `pets/createOrUpdatePetForm` | `owner/PetController.java:100` | C2 | E10 |
| `POST /owners/{ownerId}/pets/new` | `redirect:/owners/{ownerId}` (or re-render form) | `owner/PetController.java:107-136` | C2 | E11 |
| `GET /owners/{ownerId}/pets/{petId}/edit` | view `pets/createOrUpdatePetForm` | `owner/PetController.java:139` | C2 | E12 |
| `POST /owners/{ownerId}/pets/{petId}/edit` | `redirect:/owners/{ownerId}` (or re-render form) | `owner/PetController.java:144-178` | C2 | E13 |
| `GET /owners/{ownerId}/pets/{petId}/visits/new` | view `pets/createOrUpdateVisitForm` | `owner/VisitController.java:90` | C3 | E14 |
| `POST /owners/{ownerId}/pets/{petId}/visits/new` | `redirect:/owners/{ownerId}` (or re-render form) | `owner/VisitController.java:97-111` | C3 | E15 |
| `GET /vets.html` | view `vets/vetList` | `vet/VetController.java:44-56` | C4, C5 | E16 |
| `GET /vets` | `@ResponseBody Vets` (JSON/XML) — **not linked from any template** | `vet/VetController.java:65` | C4 | E17 |
| `GET /actuator/**` | framework JSON — **not linked from any template** | Actuator (framework) | C7 | E18 |

### Route adjacency map (which route links to which)

Edges below are read from template `th:href`/`th:action` and controller `redirect:` returns. The
**navbar** (`fragments/layout.html`) is included on every page, so it contributes a *global* edge
from **every screen** to its four targets.

**Global navbar edges (from every screen — `fragments/layout.html`):**
- → `GET /` (home) — `layout.html:41`
- → `GET /owners/find` — `layout.html:46`
- → `GET /vets.html` — `layout.html:51`
- → `GET /oups` (error demo) — `layout.html:56`

**In-page link / form edges:**
- `owners/findOwners.html:9` — form `th:action="@{/owners}" method=get` → `GET /owners` (search submit)
- `owners/findOwners.html:29` — link → `GET /owners/new` (Add Owner)
- `owners/ownersList.html:22` — link → `GET /owners/{id}` (open owner from results)
- `owners/ownersList.html:35-54` — pagination links → `GET /owners?page=N` (self)
- `owners/ownerDetails.html:36` — link → `GET /owners/{id}/edit` (Edit Owner)
- `owners/ownerDetails.html:38` — link → `GET /owners/{id}/pets/new` (Add New Pet)
- `owners/ownerDetails.html:72` — link → `GET /owners/{id}/pets/{petId}/edit` (Edit Pet)
- `owners/ownerDetails.html:73` — link → `GET /owners/{id}/pets/{petId}/visits/new` (Add Visit)
- `vets/vetList.html:30-50` — pagination links → `GET /vets.html?page=N` (self)

**Controller `redirect:` edges (post-submit navigation; each is a test-confirmed edge):**
- `POST /owners/new` → `GET /owners/{id}` — `OwnerController.java:86` (test `OwnerControllerTests.java:117-122`)
- `GET /owners` (single result) → `GET /owners/{id}` — `OwnerController.java:117` (test `OwnerControllerTests.java:156-157`)
- `POST /owners/{id}/edit` → `GET /owners/{id}` — `OwnerController.java:161` (test `OwnerControllerTests.java:219-220`); id-mismatch → `GET /owners/{id}/edit` — `:155` (test `:273-275`)
- `POST /owners/{id}/pets/new` → `GET /owners/{id}` — `PetController.java:136` (test `PetControllerTests.java:103-104`)
- `POST /owners/{id}/pets/{petId}/edit` → `GET /owners/{id}` — `PetController.java:178` (test `PetControllerTests.java:197-198`)
- `POST /owners/{id}/pets/{petId}/visits/new` → `GET /owners/{id}` — `VisitController.java:111` (test `VisitControllerTests.java:82-83`)

### Entry and terminal points

- **Entry points** (reachable directly by URL / from the global navbar, without navigating from
  another domain route): `GET /` (home — the primary entry), `GET /owners/find`, `GET /vets.html`,
  `GET /oups`. `GET /vets` (JSON, E17) and `GET /actuator/**` (E18) are URL-reachable but have **no**
  template link and are not part of any UI journey (api-inventory G1).
- **Hub route:** `GET /owners/{ownerId}` (owner details) is the central hub — it is the fan-out point
  to pet/visit/edit actions **and** the redirect landing target after every owner/pet/visit create or
  edit. It is neither a pure entry nor a pure terminal.
- **Terminal points** (no outbound edge except the global navbar and self-pagination):
  `GET /vets.html` (only self-pagination + navbar), the `error` view reached from `GET /oups`, and —
  as *flow* terminals — the redirect back to `GET /owners/{ownerId}` that ends every create/edit/visit
  action (the confirmation of a completed sub-journey).

### The ordered JMeter navigation scenario (strongest sequenced evidence)

`src/test/jmeter/petclinic_test_plan.jmx` defines a **single ordered `ThreadGroup`** ("User threads")
whose HTTP samplers execute in this sequence (static assets omitted):

1. `GET /` — "Home page" (sampler line 113)
2. `GET /vets.html` — "Vets" (line 182)
3. `GET /owners/find` — "Find owner" (line 205)
4. `GET /owners?lastName=` — "Find owner with lastname=\"\"" (line 228)
5. `GET /owners/{count}` — "Owner" (line 251)
6. `GET /owners/{count}/edit` — "Edit Owner" (line 274)
7. `POST /owners/{count}/edit` — "POST Edit Owner" (line 333)
8. `GET /owners/{count}/pets/new` — "New Pet" (line 356)
9. `POST /owners/{count}/pets/new` — "POST new pet" (line 401)
10. `GET /owners/{count}/pets/{petCount}/visits/new` — visit form (line 422)
11. `POST /owners/{count}/pets/{petCount}/visits/new` — POST visit (line 459)

This is a human-authored, ordered traversal of the clinic UI and is the basis for marking journeys
J1–J3, J5, and J7 **Confirmed** below. It is a load-test scenario (not a browser assertion suite), so
it evidences the *sequence of routes visited* but not per-page UI assertions — confidence is therefore
**high on the sequence, medium on intent** (it exercises paths rather than asserting a user goal).

## Step 3 & 4 — Capability hand-offs and shared state

Each route step maps to its owning capability via capability-baseline. A **hand-off** is a transition
where the owning capability changes between consecutive steps. Because C1–C3 all live in the `owner`
Java package and persist through the **owner aggregate** (capability-baseline §2), the C1↔C2 and
C1↔C3 hand-offs are *logical* capability boundaries within one deployable, carried entirely by **URL
path variables** — there is no session, token, event, or client-side store crossing them (no Spring
Security, no client JS state; ui-inventory "Frontend state management", api-inventory "Auth note").

**Owning repo / package per capability** (all in the single repo `spring-petclinic`, deployable
`petclinic`; "owner" here is the Java **package** that realizes the capability, per capability-baseline
§2 — there are no separate services):

| Capability | Name | Owning repo → package |
|-----------|------|-----------------------|
| C1 | Pet Owner Management | `spring-petclinic` → `owner` package (`OwnerController`) |
| C2 | Pet & Pet-Type Management | `spring-petclinic` → `owner` package (`PetController`) |
| C3 | Veterinary Visit Recording | `spring-petclinic` → `owner` package (`VisitController`) |
| C4 | Veterinarian & Specialty Directory | `spring-petclinic` → `vet` package (`VetController`) |
| C5 | Clinic Web Presentation & Localization | `spring-petclinic` → `system` package (`WelcomeController`, `WebConfiguration`) + `templates/` |
| C7 | Platform Operations & Observability | `spring-petclinic` → `system` package (`CrashController`) + Actuator/k8s |

Each **From → To** in the hand-off table below resolves its owning repo/package through this mapping;
every hand-off is therefore intra-repo (a package boundary within `petclinic`), not a cross-service
call.

| Hand-off (From → To) | Route transition | Shared state carried | Mechanism / evidence |
|----------------------|------------------|----------------------|----------------------|
| **C5 → C4** (Presentation → Vet Directory) | `GET /` → `GET /vets.html` | none | Clean boundary — vet list fetches its own data via `@Cacheable("vets")` `VetRepository.findAll` on entry; no param passed. Navbar link `layout.html:51` |
| **C5 → C1** (Presentation → Owner Mgmt) | `GET /` → `GET /owners/find` | none | Clean boundary — search form rendered empty. Navbar link `layout.html:46` |
| **C1 → C1** (search → results → detail) | `GET /owners/find` → `GET /owners?lastName=…` → `GET /owners/{id}` | `lastName` (query), `page` (query), `ownerId` (path) | `findOwners.html:9` form GET carries `lastName`; `ownersList.html:22` link carries `ownerId`; `OwnerController.java:94-121`. Same capability — not a hand-off |
| **C1 → C2** (Owner detail → Pet form) | `GET /owners/{id}` → `GET /owners/{id}/pets/new` (or `/pets/{petId}/edit`) | `ownerId` (path); `petId` (path, edit only) | `ownerDetails.html:38,72`. On entry, pet form loads `types` (all `PetType`) as a model attribute and (edit) the pet — API `E10`/`E12`. `PetController.java:48` (`@RequestMapping("/owners/{ownerId}")`), `:100,139` |
| **C2 → C1** (Pet form → Owner detail) | `POST /owners/{id}/pets/**` → `GET /owners/{id}` | `ownerId` (path) | `redirect:/owners/{ownerId}` `PetController.java:136,178`; test-confirmed edge |
| **C1 → C3** (Owner detail → Visit form) | `GET /owners/{id}` → `GET /owners/{id}/pets/{petId}/visits/new` | `ownerId` (path), `petId` (path) | `ownerDetails.html:73`. On entry the visit form loads the owner + pet into the model (`E14`); `VisitController.java:57,90` |
| **C3 → C1** (Visit form → Owner detail) | `POST …/visits/new` → `GET /owners/{id}` | `ownerId` (path) | `redirect:/owners/{ownerId}` `VisitController.java:111`; test-confirmed edge |
| **C5 → C5/C7** (Presentation → Error/Ops) | any screen → `GET /oups` | none | Navbar link `layout.html:56`; `CrashController.java:31` throws → default error view |

**Shared-state characterization.** Every cross-capability hand-off carries state **only as URL path
variables** (`ownerId`, `petId`), never as session/auth context, form state in storage, or an event.
The receiving route re-fetches all entities from the database on entry (owner via
`@ModelAttribute` loaders, pet types via `PetTypeRepository`), so the boundary is a **clean,
stateless, ID-only hand-off**. No `localStorage`/`sessionStorage`/embedded-JSON handoff exists
(searched; ui-inventory confirms no client JS state). No route maps to **"capability unknown"** — all
navigable routes map to C1–C5/C7 per the capability baseline.

## Step 5 — Findings

### Journey catalogue

Eight journeys are evidenced. J1–J3, J5, J7 are **Confirmed** (their sequence appears in the ordered
JMeter scenario); J4, J6 are **Partial** (final redirect edge test-confirmed, entry link inferred);
J8 is **Inferred/Partial** (navbar link inferred, the throw edge test-confirmed). A ninth composite
is described (JC0) — the full JMeter traversal — as the umbrella of J1–J3, J5, J7.

---

**Journey J1 — Browse the veterinarian directory**
- **Entry point:** `GET /` (home)
- **Exit point:** `GET /vets.html` (terminal — only self-pagination + navbar outbound)
- **Step sequence:** `GET /` *(C5)* → `GET /vets.html` *(C4)* → `GET /vets.html?page=N` *(C4, optional pagination)*
- **Capabilities crossed:** C5 (Clinic Web Presentation) → C4 (Veterinarian & Specialty Directory)
- **Hand-off points:** C5 → C4 at step 1→2 (`/` → `/vets.html`)
- **Shared state at hand-off:** none — clean boundary; the vet list fetches its own cached data on entry
- **Evidence type:** Confirmed (JMeter `petclinic_test_plan.jmx` samplers "Home page"→"Vets"); pagination inferred from `vetList.html:30-50`
- **Confidence:** high
- **Evidence:** `src/test/jmeter/petclinic_test_plan.jmx:113,182`; `fragments/layout.html:51`; `vet/VetController.java:44-56`; `vets/vetList.html:30-50`

**Journey J2 — Find an owner and view their record**
- **Entry point:** `GET /` (home) → typically via `GET /owners/find`
- **Exit point:** `GET /owners/{ownerId}` (owner-details hub)
- **Step sequence:** `GET /` *(C5)* → `GET /owners/find` *(C1)* → `GET /owners?lastName=…` *(C1)* → `GET /owners/{ownerId}` *(C1)*
- **Capabilities crossed:** C5 → C1 (Pet Owner Management)
- **Hand-off points:** C5 → C1 at step 1→2 (`/` → `/owners/find`)
- **Shared state at hand-off:** none at C5→C1; within C1: `lastName` (query), `page` (query), `ownerId` (path) carried between search and detail. A single search hit **redirects straight to** `/owners/{ownerId}` (`OwnerController.java:117`)
- **Evidence type:** Confirmed (JMeter "Home"→"Find owner"→"Find owner lastname=\"\""→"Owner"); single-result redirect is a test-confirmed edge (`OwnerControllerTests.java:156-157`)
- **Confidence:** high
- **Evidence:** `petclinic_test_plan.jmx:113,205,228,251`; `owner/OwnerController.java:89-121,169`; `owners/findOwners.html:9`; `owners/ownersList.html:22`

**Journey J3 — Edit an existing owner**
- **Entry point:** `GET /owners/{ownerId}` (from J2)
- **Exit point:** `GET /owners/{ownerId}` (redirect back after save)
- **Step sequence:** `GET /owners/{ownerId}` *(C1)* → `GET /owners/{ownerId}/edit` *(C1)* → `POST /owners/{ownerId}/edit` *(C1)* → `redirect:/owners/{ownerId}` *(C1)*
- **Capabilities crossed:** C1 only — **no cross-capability hand-off**
- **Hand-off points:** none (single-capability journey)
- **Shared state at hand-off:** `ownerId` (path) carried across all steps; on id-mismatch the controller redirects back to the edit form (`OwnerController.java:155`)
- **Evidence type:** Confirmed (JMeter "Owner"→"Edit Owner"→"POST Edit Owner"); redirect edge test-confirmed (`OwnerControllerTests.java:219-220,273-275`)
- **Confidence:** high
- **Evidence:** `petclinic_test_plan.jmx:251,274,333`; `owner/OwnerController.java:139-161`; `owners/ownerDetails.html:36`

**Journey J4 — Register a new owner**
- **Entry point:** `GET /owners/find`
- **Exit point:** `GET /owners/{ownerId}` (redirect to the new owner's detail)
- **Step sequence:** `GET /owners/find` *(C1)* → `GET /owners/new` *(C1)* → `POST /owners/new` *(C1)* → `redirect:/owners/{ownerId}` *(C1)*
- **Capabilities crossed:** C1 only — **no cross-capability hand-off**
- **Hand-off points:** none (single-capability journey)
- **Shared state at hand-off:** new `ownerId` generated on save and carried into the redirect URL (`OwnerController.java:86`); no inbound state
- **Evidence type:** Partial — the `find → new` link is **inferred from router adjacency** (`findOwners.html:29`); the `POST /owners/new → /owners/{id}` redirect is a **test-confirmed edge** (`OwnerControllerTests.java:117-122`). Not present in the JMeter scenario
- **Confidence:** medium
- **Evidence:** `owners/findOwners.html:29`; `owner/OwnerController.java:72-86`

**Journey J5 — Add a pet to an owner**
- **Entry point:** `GET /owners/{ownerId}` (owner-details hub)
- **Exit point:** `GET /owners/{ownerId}` (redirect back after save)
- **Step sequence:** `GET /owners/{ownerId}` *(C1)* → `GET /owners/{ownerId}/pets/new` *(C2)* → `POST /owners/{ownerId}/pets/new` *(C2)* → `redirect:/owners/{ownerId}` *(C1)*
- **Capabilities crossed:** C1 (Pet Owner Mgmt) ↔ C2 (Pet & Pet-Type Mgmt)
- **Hand-off points:** C1 → C2 at step 1→2 (`/owners/{id}` → `/owners/{id}/pets/new`); C2 → C1 at step 3→4 (redirect)
- **Shared state at hand-off:** `ownerId` (path) into the pet form; the form loads all `PetType` (`types`) on entry (data fetch, api-inventory E10). Return carries `ownerId` in the redirect
- **Evidence type:** Confirmed (JMeter "New Pet"→"POST new pet"); redirect edge test-confirmed (`PetControllerTests.java:103-104`)
- **Confidence:** high
- **Evidence:** `petclinic_test_plan.jmx:356,401`; `owner/PetController.java:48,100-136`; `owners/ownerDetails.html:38`

**Journey J6 — Edit an existing pet**
- **Entry point:** `GET /owners/{ownerId}` (owner-details hub)
- **Exit point:** `GET /owners/{ownerId}` (redirect back after save)
- **Step sequence:** `GET /owners/{ownerId}` *(C1)* → `GET /owners/{ownerId}/pets/{petId}/edit` *(C2)* → `POST /owners/{ownerId}/pets/{petId}/edit` *(C2)* → `redirect:/owners/{ownerId}` *(C1)*
- **Capabilities crossed:** C1 ↔ C2
- **Hand-off points:** C1 → C2 at step 1→2; C2 → C1 at step 3→4 (redirect)
- **Shared state at hand-off:** `ownerId` + `petId` (path) into the edit form; `types` loaded on entry (E12). Return carries `ownerId`
- **Evidence type:** Partial — the `ownerDetails → pet edit` link is **inferred from router adjacency** (`ownerDetails.html:72`); the redirect is a **test-confirmed edge** (`PetControllerTests.java:197-198`). Not in the JMeter scenario
- **Confidence:** medium
- **Evidence:** `owners/ownerDetails.html:72`; `owner/PetController.java:139-178`

**Journey J7 — Record a veterinary visit for a pet**
- **Entry point:** `GET /owners/{ownerId}` (owner-details hub)
- **Exit point:** `GET /owners/{ownerId}` (redirect back after save)
- **Step sequence:** `GET /owners/{ownerId}` *(C1)* → `GET /owners/{ownerId}/pets/{petId}/visits/new` *(C3)* → `POST /owners/{ownerId}/pets/{petId}/visits/new` *(C3)* → `redirect:/owners/{ownerId}` *(C1)*
- **Capabilities crossed:** C1 (Pet Owner Mgmt) ↔ C3 (Veterinary Visit Recording)
- **Hand-off points:** C1 → C3 at step 1→2 (`/owners/{id}` → `.../visits/new`); C3 → C1 at step 3→4 (redirect)
- **Shared state at hand-off:** `ownerId` + `petId` (path) into the visit form; the form loads owner + pet into the model on entry (E14). Return carries `ownerId`
- **Evidence type:** Confirmed (JMeter visit-form GET→POST, lines 422,459); redirect edge test-confirmed (`VisitControllerTests.java:82-83`)
- **Confidence:** high
- **Evidence:** `petclinic_test_plan.jmx:422,459`; `owner/VisitController.java:57,90-111`; `owners/ownerDetails.html:73`

**Journey J8 — Trigger the error-handling demo**
- **Entry point:** any screen (global navbar)
- **Exit point:** default `error` view (HTTP 500) — terminal
- **Step sequence:** *(any screen, e.g. C5)* → `GET /oups` *(C5/C7)* → `error` view *(C5/C7)*
- **Capabilities crossed:** C5 (Presentation) → C7 (Platform Operations — deliberate crash demo)
- **Hand-off points:** C5 → C5/C7 at the navbar → `/oups` transition
- **Shared state at hand-off:** none
- **Evidence type:** Partial/Inferred — the navbar `/oups` link is **inferred from router adjacency** (`layout.html:56`); that `/oups` throws to the error view is a **test-confirmed edge** (`CrashControllerTests` / `CrashControllerIntegrationTests`)
- **Confidence:** medium
- **Evidence:** `fragments/layout.html:56`; `system/CrashController.java:31`; `src/test/java/.../system/CrashControllerIntegrationTests.java`

**Composite JC0 — Full clinic-management session (JMeter macro-traversal)**
- The JMeter plan executes J1 → J2 → J3 → J5 → J7 as one continuous ordered thread (Home → Vets →
  Find → Owner → Edit Owner → Add Pet → Record Visit). This is the single most complete confirmed
  navigation sequence in the repo; the per-journey rows above decompose it plus add the
  adjacency-only J4/J6/J8.
- **Evidence type:** Confirmed · **Confidence:** high · **Evidence:** `src/test/jmeter/petclinic_test_plan.jmx` (single ordered `ThreadGroup`)

### Cross-capability navigation summary

| From Capability | To Capability | Journeys Using This Edge | Evidence |
|-----------------|---------------|--------------------------|----------|
| C5 Presentation | C4 Vet Directory | J1 | `fragments/layout.html:51`; `petclinic_test_plan.jmx:182` |
| C5 Presentation | C1 Owner Mgmt | J2, J4 (via `/owners/find`) | `fragments/layout.html:46`; `petclinic_test_plan.jmx:205` |
| C1 Owner Mgmt | C2 Pet Mgmt | J5, J6 | `owners/ownerDetails.html:38,72`; `owner/PetController.java:48` |
| C2 Pet Mgmt | C1 Owner Mgmt | J5, J6 (post-save redirect) | `owner/PetController.java:136,178` |
| C1 Owner Mgmt | C3 Visit Recording | J7 | `owners/ownerDetails.html:73`; `petclinic_test_plan.jmx:422` |
| C3 Visit Recording | C1 Owner Mgmt | J7 (post-save redirect) | `owner/VisitController.java:111` |
| C5 Presentation | C5/C7 Ops (error demo) | J8 | `fragments/layout.html:56`; `system/CrashController.java:31` |
| C1 Owner Mgmt | C1 Owner Mgmt | J2, J3, J4 (search→detail→edit; internal) | `owners/ownersList.html:22`; `owner/OwnerController.java:117,161` |
| C4 Vet Directory | C4 Vet Directory | J1 (pagination; internal) | `vets/vetList.html:30-50` |

> **Global navbar note.** Because `fragments/layout.html` renders on every page, each screen also has
> outbound navbar edges to `/` (C5), `/owners/find` (C1), `/vets.html` (C4), and `/oups` (C5/C7).
> These are omitted per-row above to avoid an all-to-four explosion; they mean any capability can
> transition back to C5/C1/C4 via the navbar. Evidence: `fragments/layout.html:41-60`.

### Shared state summary

| Hand-off (From → To) | State Carried | Mechanism | Evidence |
|----------------------|---------------|-----------|----------|
| C5 → C4 (`/` → `/vets.html`) | none — clean boundary | — (vet list re-fetches own cached data) | `vet/VetController.java:44`; `vet/VetRepository.java:45` |
| C5 → C1 (`/` → `/owners/find`) | none — clean boundary | — (empty search form) | `owner/OwnerController.java:89` |
| C1 → C2 (`/owners/{id}` → `/owners/{id}/pets/new`\|`/pets/{petId}/edit`) | `ownerId`; `petId` (edit) | URL path variable; `types` fetched on entry | `owner/PetController.java:48,100,139`; api-inventory E10/E12 |
| C1 → C3 (`/owners/{id}` → `.../visits/new`) | `ownerId`, `petId` | URL path variable; owner+pet fetched on entry | `owner/VisitController.java:57,90`; api-inventory E14 |
| C2/C3 → C1 (post-save redirect) | `ownerId` | server `redirect:` URL path variable | `owner/PetController.java:136,178`; `owner/VisitController.java:111` |
| C5 → C5/C7 (`/oups`) | none | — | `system/CrashController.java:31` |

> **No session/auth/event state crosses any boundary.** There is no Spring Security (api-inventory
> "Auth note", G3), no client-side store (ui-inventory "Frontend state management"), and no event bus
> (api-inventory Part 3). All cross-capability state is ID-only, carried in the URL path/query.

### Evidence quality

- **Confirmed journeys** (sequence evidenced in the ordered JMeter scenario): **J1** (Browse vets),
  **J2** (Find owner & view), **J3** (Edit owner), **J5** (Add pet), **J7** (Record visit), and the
  composite **JC0** (full session). Redirect edges within these are additionally verified by MockMvc
  controller tests.
- **Inferred / Partial journeys** (router adjacency + a test-confirmed final edge, no ordered-sequence
  coverage): **J4** (Register new owner — `[partial]`), **J6** (Edit pet — `[partial]`), **J8** (Error
  demo — `[partial]`).
- **Gaps** — journeys that could plausibly exist by capability ownership but have **no** navigation
  evidence:
  - `[likely gap]` A **consumer journey for the `GET /vets` JSON endpoint** (E17, C4): the endpoint
    exists but no template links to it and no in-repo client calls it (api-inventory G1). No UI journey
    reaches it.
  - `[likely gap]` An **`/actuator/**` operations journey** (E18, C7): reachable by URL but not linked
    from any UI navigation; no human-authored sequence visits it.
  - `[not observed]` Any **pet/owner deletion, appointment scheduling, authentication/login, or
    multi-owner batch** journey — no such routes exist in code (the CAKE scheduling/auth journeys are
    Future/Intended State, Not Implemented).

### Confidence assessment

| Journey | Confidence | Key Uncertainty | Evidence Basis |
|---------|------------|-----------------|----------------|
| J1 Browse vets | high | JMeter evidences the path, not a user-goal assertion | JMeter scenario + navbar link + controller |
| J2 Find owner & view | high | Real user may paginate `ownersList` before opening a detail (JMeter uses empty-lastName path) | JMeter scenario + template links + controller/test |
| J3 Edit owner | high | None material | JMeter scenario + template link + redirect test |
| J4 Register new owner | medium | Entry link inferred; not in the ordered scenario | Template adjacency + POST-redirect test |
| J5 Add pet | high | None material | JMeter scenario + template link + redirect test |
| J6 Edit pet | medium | Entry link inferred; not in the ordered scenario | Template adjacency + POST-redirect test |
| J7 Record visit | high | None material | JMeter scenario + template link + redirect test |
| J8 Error demo | medium | Navbar link inferred; intent is a demo, not a user goal | Navbar adjacency + crash-controller test |
| JC0 Full session | high | Load-test scenario (sequence high, per-step intent medium) | Single ordered JMeter ThreadGroup |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The `petclinic_test_plan.jmx` ordered ThreadGroup represents a realistic user navigation sequence and is valid "confirmed" journey evidence. | It is the only human-authored *ordered* multi-route asset in the repo; samplers run top-to-bottom in one thread group. | J1–J3/J5/J7 would drop to "inferred" if the plan is deemed a synthetic load probe rather than a journey. | Confirm with QA/perf owner whether the JMeter order reflects an intended user flow. |
| A2 | Cross-capability hand-offs carry state **only** as URL path/query variables (no hidden session/token/storage handoff). | No Spring Security, no client JS state, no event bus were found (baselines + code search). | Impact analysis would understate coupling if an out-of-band handoff exists. | Runtime trace of a create/edit request; confirm no session attributes are read by receiving controllers. |
| A3 | J4 (register owner) and J6 (edit pet) entry links are exercised by users despite absence from the ordered scenario. | The template links (`findOwners.html:29`, `ownerDetails.html:72`) and post-save redirects are present and test-confirmed. | These journeys may be mis-prioritized if rarely used. | Add an e2e/perf step covering them, or confirm via access logs. |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Does the CAKE scheduling/EMR journey + persona model (front-desk staff, appointment booking, provider selection) represent planned future scope for this platform, or an unrelated catalog for a different product? | Determines whether an entire set of appointment-scheduling journeys is future scope vs out-of-scope noise for this repo. | Not implemented in code; recorded as Future/Intended State (Not Implemented). | Product / architecture owner |
| Q2 | Is there any consumer journey for the `GET /vets` JSON endpoint (E17) and the `/actuator/**` surface (E18)? | Both are URL-reachable but linked from no UI navigation; determines whether hidden (external/ops) journeys exist. | Unknown — no in-repo template or client reaches them (api-inventory G1/G2). | Platform / API owner |
| Q3 | Should browser-driven e2e coverage (Cypress/Playwright/Selenium) be added, given only a JMeter load plan and per-edge MockMvc tests exist today? | Confirmed-journey confidence rests on a load plan; real UI assertions would raise it and cover J4/J6/J8. | None present today; JMeter + MockMvc are the current substitutes. | QA / platform owner |
