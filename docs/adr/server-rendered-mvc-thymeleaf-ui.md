# ADR: Server-Rendered MVC (Thymeleaf) over Client-Side SPA/MFE

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-002 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Integration · **Impact:** high · **Dependencies:** [single-deployable-layered-monolith](single-deployable-layered-monolith.md) (ADR-CANDIDATE-001)

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The entire presentation tier is **server-rendered** — Spring MVC controllers return Thymeleaf view
names and the server produces the full HTML page per request. There is no single-page application,
no micro-frontend, no client-side JS framework, and no JavaScript build.

- ✅ Controllers under `owner/`, `vet/`, and `system/` return template view names — e.g.
  `VetController.showVetList` returns `"vets/vetList"` (`vet/VetController.java:56`), with views under
  `src/main/resources/templates/**` and shared chrome in `templates/fragments/layout.html`
  (architecture-baseline §4).
- ✅ There is **no** `package.json`, npm/webpack/Vite build, and no React/Angular/Vue/Svelte
  dependency anywhere in the repo; styling is Bootstrap 5.3.8 + font-awesome 4.7.0 delivered as
  **WebJars**, with SCSS compiled to `petclinic.css` via libsass at build time (architecture-baseline
  §4).
- ✅ A single content-negotiated `@ResponseBody` endpoint (`GET /vets`) returns the `Vets` wrapper as
  JSON (`vet/VetController.java:65–71`) — the only machine-facing surface; it does not change the UI
  delivery model.

Choosing server-side MPA over a SPA/MFE is an architecturally significant, durable decision about the
entire presentation tier: it shapes the delivery model, SEO, client-state handling, testing approach,
and required team skills. It is bounded by the single-deployable monolith decision
(ADR-CANDIDATE-001): the UI is served from the same process that serves the domain.

The CAKE catalog's Webpack Module Federation micro-frontend approach (`@webpt/scheduling-mfe`,
`ADR-PLATFORM-021/024/041`, host-shell/BFF integration) belongs to the WebPT EMR scheduling program —
a different platform — and is **not** realized here (Source-of-Truth: code wins).

## Decision

**Render the UI on the server with Spring MVC controllers returning Thymeleaf templates** (a
server-side multi-page application), delivering CSS/JS assets as bundled WebJars — rather than
building a client-side SPA over a JSON API or a micro-frontend composed via Module Federation. A
single `@ResponseBody` JSON endpoint (`/vets`) is retained for machine consumers, but is not the
primary delivery model.

## Alternatives Considered

1. **Client-side SPA (React/Angular/Vue) over a JSON API.**
   *Rejected* — would require a JavaScript build toolchain, a separate client-state model, a full JSON
   API surface for every workflow, and front-end specialist skills, in exchange for richer client
   interactivity the veterinary-clinic CRUD workflows do not need. Server-rendered pages deliver the
   forms-and-lists UX directly with no client framework and with SEO/first-paint for free.

2. **Micro-frontends via Webpack Module Federation (the CAKE-catalog scheduling approach).**
   *Rejected* — module-federated MFEs with a host shell, shared React singletons, and BFF pass-through
   solve a multi-team, multi-app composition problem that does not exist in a single-deployable,
   single-team monolith. The operational and integration complexity is unjustified here.

3. **Server-rendered HTML with a progressive-enhancement JS layer (e.g. htmx / Turbo).**
   *Rejected (as an unnecessary addition)* — plain Thymeleaf with Post-Redirect-Get already meets the
   interaction needs; adding a partial-update library introduces a dependency and build step without a
   demonstrated UX gap. 🎯 A reasonable future enhancement if richer in-page interactivity is required.

## Consequences

**Positive**
- ✅ No JavaScript build, no client framework to maintain, no client/server contract drift — the
  server owns rendering end-to-end.
- SEO-friendly, fast first paint, and simple end-to-end testing (request → rendered HTML).
- All user/session context (including locale via `SessionLocaleResolver`) lives server-side; there is
  no client state store or hydration payload to manage (architecture-baseline §4).

**Negative / trade-offs**
- Rich, app-like client interactivity requires full-page navigation or a later JS addition; the model
  is form-and-reload, not real-time.
- Presentation logic is coupled to the deployable (bounded by ADR-CANDIDATE-001) — the UI cannot be
  deployed or scaled independently of the backend.
- The lone JSON endpoint (`/vets`) is an inconsistent API surface (one endpoint, not a general API),
  which limits programmatic integration.

**Pattern relationship**
- **Instantiates** *Server-Rendered MVC with Template Views*
  (`docs/architecture-inventory/patterns/ui/server-rendered-mvc-views.md`, Candidate) and composes
  with *Post-Redirect-Get Form Handling with Bean Validation*, *Request-Scoped Internationalization*,
  and *Content-Negotiated Dual Representation* (the `/vets` JSON case). No **Pattern Drift**.
- **Pattern Update Proposal:** none — the existing UI patterns already capture this approach; promote
  them to Approved alongside this ADR so any future SPA/MFE PR is flagged as drift.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Controllers return Thymeleaf view names | ✅ | `vet/VetController.java:56` (`return "vets/vetList"`); `src/main/resources/templates/**` |
| E2 | No JS framework / package.json / npm build | ✅ | repo root (no `package.json`); architecture-baseline §4 |
| E3 | WebJars assets + SCSS→CSS at build time | ✅ | `pom.xml` (WebJars, libsass), `src/main/scss/*`; architecture-baseline §4 |
| E4 | Single JSON endpoint via `@ResponseBody` | ✅ | `vet/VetController.java:65–71` (`GET /vets` returns `Vets`) |
| E5 | CAKE MFE/Module-Federation ADRs govern a different platform | ✅ | `cake_graph_query` (2026-08-18): `ADR-PLATFORM-021/024/041` (`@webpt/scheduling-mfe`) — absent from this tree |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Is richer client-side interactivity (partial updates / real-time) a foreseeable requirement? | Would drive a progressive-enhancement (htmx/Turbo) or SPA evolution and re-open this decision. | 🎯 Not currently; add progressive enhancement before a full SPA if a gap appears. | Product / architecture owner |
| Q2 | Should the machine-facing surface grow beyond the single `/vets` JSON endpoint into a general API? | Determines whether a SPA/MFE or external-integration path becomes viable. | — (no general API today). | Platform architecture owner |
