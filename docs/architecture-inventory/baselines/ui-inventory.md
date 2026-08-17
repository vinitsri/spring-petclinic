# UI Inventory Baseline — Petclinic Platform (Evidence-Based)

- **Project Key:** VE
- **Project Name:** Petclinic platform
- **Stage:** Architecture discovery — UI / frontend inventory (observed current-state only)
- **Branch:** `arch-discovery-57f5f6e8-e25d-475f-8aeb-42d3fc5622b6`
- **Date:** 2026-08-17
- **Scope:** User-facing / presentation assets of the single `petclinic` deployable — server-rendered
  templates, static CSS/JS/font assets, the CSS build pipeline, and any client-side application
  surface — across the one repository in this workspace.

> **Confidence legend:** `high` = directly evidenced by source code, template, or config ·
> `medium` = evidenced but with an inference step · `low` = weak/naming-only signal ·
> `unknown` = insufficient evidence.
>
> **Status labels:** **observed** = read directly from a template/asset/config · **inferred** =
> derived from framework behaviour or naming · **not observed** = searched for and not present ·
> **Future/Intended State (Not Implemented)** = present in catalog narrative but absent from code.

## Method and grounding

Evidence basis: direct walk of the `spring-petclinic` source tree — `src/main/resources/templates/**`
(Thymeleaf views), `src/main/resources/static/resources/**` (compiled CSS, fonts, images),
`src/main/scss/**` (SCSS sources), `pom.xml` / `build.gradle` (build plugins and WebJar
dependencies), `.github/workflows/**` (CI), `docker-compose.yml`, `README.md`, and the prior
baselines `docs/architecture-inventory/baselines/api-inventory.md` and
`docs/architecture-inventory/baselines/capability-baseline.md`. Endpoint identifiers (E1–E18) and
capability identifiers (C1–C7) referenced below are defined in those two baselines and are used as
cross-references rather than re-derived from template/JS scanning alone.

The workspace contains **one** repository, `spring-petclinic` (git URL
`https://github.com/vinitsri/spring-petclinic`, base ref `feature/VE-216/aidlc`), a single-deployable
Spring Boot monolith (Kubernetes/Maven deployable name `petclinic`). It is a **server-rendered Spring
MVC + Thymeleaf** application: the presentation layer is HTML templates rendered on the server, styled
by one compiled CSS file and vendor CSS/JS delivered through WebJars. There is **no** JavaScript
application, **no** SPA, **no** micro-frontend, and **no** Node/npm frontend toolchain. These are
demonstrated below by enumerating every directory checked, not asserted.

### Source-of-truth conflict — CAKE catalog vs observed code (MANDATORY surfacing)

Consistent with the API and capability baselines, the CAKE catalog for the working tenant
(`5K4DVCTX`) describes a **different platform's** frontend — a WebPT EMR appointment-scheduling
program. The graph returns a `Scheduling MFE` (React Micro Frontend), a `Portal Access`
SystemComponent under a "Patient Engagement Context", and enterprise UI-adjacent principles
(`Security by Design`, `Privacy by Design`, `Cloud-Native Design`, DDD bounded contexts). No
PetClinic-specific UI surface, screen, route, or design-system node was returned.

- **What the catalog implies:** a React Micro Frontend (`Scheduling MFE`) composed into a patient
  portal — i.e. a client-side SPA/MFE presentation tier.
- **What the source code shows:** none of it exists in `spring-petclinic`. There is no React, no MFE
  composition mechanism, no `package.json`, no JS bundle, and no client-side router. The presentation
  tier is Thymeleaf server-side HTML with Bootstrap/Font-Awesome vendor assets (verified across
  `src/main/resources/templates/**`, `src/main/resources/static/**`, `src/main/scss/**`, `pom.xml`).
- **Resolution (code wins):** per the Source-of-Truth rule, source code is authoritative. The CAKE
  `Scheduling MFE` / patient-portal frontend model is recorded as **Future/Intended State (Not
  Implemented)** for this repository and is **not** entered as an observed UI surface, framework, or
  MFE pattern below. Whether that program is planned scope is `[unknown]` — see ABQ.

CAKE is treated as background narrative context only. It did not add or remove any UI surface,
component, route, or asset in this inventory; every finding below is grounded in code/config.

## Directories and patterns checked (absence is demonstrated, not assumed)

A single repo (`spring-petclinic`) is in scope. The following were walked to completion:

| Search target | Path(s) / glob(s) inspected | Result |
|---------------|------------------------------|--------|
| View templates | `src/main/resources/templates/**/*.html` | **12 Thymeleaf HTML templates found** (see Routes/screens) |
| Static web assets | `src/main/resources/static/resources/{css,fonts,images}/**` | 1 CSS file, 8 font files, 5 images; **no `js/` directory, no `.js` files** |
| SCSS sources | `src/main/scss/*.scss` | 4 SCSS files (`petclinic`, `header`, `typography`, `responsive`) |
| JS framework roots | `public/`, `src/frontend/`, `src/client/`, `src/ui/`, `resources/js/`, `resources/views/`, `static/`, `assets/`, `web/`, `wwwroot/`, `webroot/` | **none exist** in this repo |
| Node manifest | `package.json` (repo root and all subtrees) | **none found** |
| JS framework/build config | `angular.json`, `next.config.js`, `nuxt.config.*`, `vite.config.*`, `webpack.config.*`, `rollup.config.*`, `Gruntfile.js`, `Gulpfile.js`, `esbuild.config.*`, `.babelrc`, `babel.config.*`, `tsconfig.json` | **none found** |
| Other view engines | `**/*.{jsx,tsx,vue,phtml,blade.php,erb,html.twig,cshtml,hbs,ejs}` | **none found** |
| Author-written JS | `**/*.js` excluding `*.min.js` (vendor) | **none found** — the only JS is one inline `<script>` block and the Bootstrap vendor bundle |
| Component-library / design-system dirs | dirs named `components/`, `ui/`, `shared/`, `lib/`, `design-system/`, `ui-kit/`, `.storybook/`, `storybook/` | **none found** |
| Storybook / token files | `*.stories.*`, `tokens.*`, `design-tokens.*`, `theme.*`, `variables.css`, `_variables.scss` | **none found** |

Every "none found" row above was produced by a filesystem walk of the whole repo (excluding `.git`
and build output), not by inspecting the repo root only.

---

## Core inventory

### Rendering model

**Server-rendered (Spring MVC + Thymeleaf). Not SPA, not MFE.** — confidence **high**.

Every view is a Thymeleaf template resolved server-side by a Spring MVC controller; the browser
receives fully-rendered HTML. Navigation is full-page GET requests and HTML form POSTs — there is no
client-side router, no client app mount/bootstrap, and no XHR/`fetch` data layer in any template.
The only client JavaScript is (1) the vendor Bootstrap bundle for the responsive navbar and (2) one
small inline `hideMessages()` DOM timer on the owner-details page. Per the SPA-classification rule,
no SPA/MFE label is applied because no client-side routing, app entry/mount, or bundled client
application tied to the domain is present.

- **Evidence (observed):** `src/main/resources/templates/**` (all views declare
  `xmlns:th="https://www.thymeleaf.org"` and `th:replace="~{fragments/layout :: layout(...)}"`);
  `templates/fragments/layout.html:69` (`<th:block th:insert="${template}" />` server-side view
  composition); controllers `owner/OwnerController.java`, `owner/PetController.java`,
  `owner/VisitController.java`, `vet/VetController.java`, `system/WelcomeController.java` (E1–E18 in
  api-inventory). Corroborated by capability-baseline C5 ("no client-side JS framework
  (server-rendered)").

### Framework and version

There is **no application-level JS framework** (no React/Angular/Vue/Svelte/Backbone/Ext). The
client-side libraries present are **vendor CSS/JS delivered via Maven WebJars**, not an app
framework, and the templating "framework" is server-side Thymeleaf.

| Library / stack | Version | ~Release era | Role (evidence-backed) | Detected vs actively used | Confidence | Evidence |
|-----------------|---------|--------------|------------------------|---------------------------|------------|----------|
| Thymeleaf | (Spring Boot-managed; version not pinned in `pom.xml`) | — | **Primary server-side view technology** — all HTML is Thymeleaf | **actively used** (every template + all controllers return view names) | high | `templates/**/*.html`; `pom.xml` `spring-boot-starter-thymeleaf` |
| Bootstrap (WebJar `org.webjars.npm:bootstrap`) | **5.3.8** | 2024–2025 | **Globally-shared vendor UI toolkit** — layout grid, navbar, form styling; its JS bundle drives the collapsible navbar | **actively used** (loaded by shared layout; SCSS `@import "bootstrap"`) | high | `pom.xml:27,104-105`; `layout.html:84` (`bootstrap.bundle.min.js`); `src/main/scss/petclinic.scss:14` |
| Font Awesome (WebJar `org.webjars.npm:font-awesome`) | **4.7.0** | 2016 | **Globally-shared vendor icon font** (CSS only, no JS) | **actively used** (icons in navbar, vet pagination, form feedback) | high | `pom.xml:28,110-111`; `layout.html:13`; `vets/vetList.html`, `fragments/inputField.html` |
| webjars-locator-lite | 1.1.3 | — | Resolves versionless `/webjars/**` URLs at runtime | actively used (enables `@{/webjars/...}` links) | high | `pom.xml:26,98-99` |

- **`package.json` / vendor-header fallback:** No `package.json` exists anywhere, so framework
  versions are taken from the **authoritative Maven dependency declarations** (`pom.xml`), which is
  stronger evidence than JS file headers. The Bootstrap/Font-Awesome JS/CSS is not committed to the
  repo; it is unpacked from the WebJar at build/runtime.
- **No SPA/MFE framework detected** — see Rendering model and MFE patterns.

### Build pipeline

**JS build pipeline: none — there is no JavaScript to build.** The only asset-generation step is a
**CSS (SCSS→CSS) compilation that is opt-in and dormant in the default build.** — confidence **high**.

- **SCSS→CSS via `libsass-maven-plugin` 0.3.4**, plus `maven-dependency-plugin` unpacking the
  Bootstrap WebJar so its SCSS is on the include path. Both executions live **inside the `css` Maven
  profile**, which is **not active by default**. Evidence: `pom.xml:323-324` (`<profile><id>css</id>`),
  `:351-369` (libsass plugin, `inputPath=src/main/scss/`, `outputPath=.../static/resources/css/`),
  `:327-350` (dependency-plugin unpack).
- **The compiled output is committed to the repo** and shipped as a static asset:
  `src/main/resources/static/resources/css/petclinic.css`. `README.md:98-100` confirms this — *"There
  is a `petclinic.css` … It was generated from the `petclinic.scss` source … If you make changes to
  the scss … you will need to re-compile … using the Maven profile 'css', i.e. `./mvnw package -P
  css`. There is no build profile for Gradle to compile the CSS."*
- **CI does not run the CSS profile.** `.github/workflows/maven-build.yml:29` runs `./mvnw -B verify`
  and `.github/workflows/gradle-build.yml:31` runs `./gradlew build` — neither passes `-P css` nor
  invokes any Node/npm/bundler step. `docker-compose.yml` defines only MySQL/PostgreSQL services; no
  `Dockerfile` exists.

**Build/runtime distinction:**
- **Detected in repo:** `libsass-maven-plugin`, `maven-dependency-plugin` (Bootstrap unpack).
- **Actively building the analyzed UI:** none in the default/CI build — the shipped `petclinic.css`
  is a pre-generated committed artifact; the SCSS toolchain only runs when a developer explicitly
  invokes `-P css`.
- **Legacy/dormant:** the `css` profile behaves as an **opt-in developer tool**, not an active
  pipeline. There is **no** webpack/vite/rollup/grunt/gulp/esbuild/babel/tsconfig anywhere (all
  searched, none found).

### MFE / micro-frontend patterns

**None detected.** — confidence **high**.

A whole-repo scan for `federation`, `mfe`, `microfrontend`, `__webpack_share_scopes__`,
`customElements.define`, `single-spa`, `loadRemoteModule`, `registerApplication`,
`defineCustomElement`, `ModuleFederationPlugin`, `remoteEntry`, `remotes:`, `exposes:`, `shared:`,
`System.import`, `importmap`/`import-map`, `web component`, `shadowRoot`, `attachShadow` across
`*.html/*.js/*.css/*.scss/*.xml/*.gradle/*.json/*.yml/*.yaml` returned **zero matches**. No
composition mechanism (Module Federation config, single-spa registration, custom-element/remote
loading, import maps) exists. The CAKE `Scheduling MFE` node belongs to the unrelated WebPT program
and is recorded as **Future/Intended State (Not Implemented)** (see the source-of-truth conflict).

### Module structure (JS modules for the domain)

There is **essentially no client-side JS module layer**. — confidence **high**.

| Script | Type | Purpose | Scope | Evidence |
|--------|------|---------|-------|----------|
| `bootstrap.bundle.min.js` | Vendor bundle (WebJar) | Drives the collapsible responsive navbar (`data-bs-toggle="collapse"`) | Global (loaded by shared layout on every page) | `layout.html:84`; `layout.html:23` |
| Inline `hideMessages()` | Page-scoped inline `<script>` | Hides `#success-message` / `#error-message` after 3s via `setTimeout` | Owner-details view only | `owners/ownerDetails.html:80-91` |

No author-written `.js` module files exist (`find … -name '*.js' -not -name '*.min.js'` → empty).
There is no ES-module graph, no bundler entrypoint, and no per-domain JS modules.

### Routes / screens

Routing is **server-side and centralized** in Spring MVC controllers; each route resolves a Thymeleaf
view. The 12 templates and their owning routes (route identifiers per api-inventory E1–E18):

| View template | Route(s) | Controller (evidence) | Capability |
|---------------|----------|------------------------|------------|
| `welcome.html` | `GET /` (E1) | `system/WelcomeController.java` | C5 |
| `error.html` | default `/error` view | Spring Boot `BasicErrorController` (framework) | C5, C7 |
| `owners/findOwners.html` | `GET /owners/find` (E5), search form → `GET /owners` (E6) | `owner/OwnerController.java` | C1 |
| `owners/ownersList.html` | `GET /owners` (E6, many results) | `owner/OwnerController.java` | C1 |
| `owners/ownerDetails.html` | `GET /owners/{ownerId}` (E7) | `owner/OwnerController.java` | C1, C2, C3 |
| `owners/createOrUpdateOwnerForm.html` | `GET/POST /owners/new` (E3/E4), `GET/POST /owners/{ownerId}/edit` (E8/E9) | `owner/OwnerController.java` | C1 |
| `pets/createOrUpdatePetForm.html` | `GET/POST /owners/{ownerId}/pets/new` (E10/E11), `.../pets/{petId}/edit` (E12/E13) | `owner/PetController.java` | C2 |
| `pets/createOrUpdateVisitForm.html` | `GET/POST /owners/{ownerId}/pets/{petId}/visits/new` (E14/E15) | `owner/VisitController.java` | C3 |
| `vets/vetList.html` | `GET /vets.html` (E16) | `vet/VetController.java` | C4 |
| `fragments/layout.html` | — (shared layout shell + `menuItem` fragment) | included by all views | C5 |
| `fragments/inputField.html` | — (reusable form-input fragment) | included by owner/pet/visit forms | C1, C2, C3 |
| `fragments/selectField.html` | — (reusable form-select fragment) | included by pet form | C2 |

- **Confidence:** high — every mapping is read directly from controller `@GetMapping`/`@PostMapping`
  return values and template `th:replace`/`th:fragment` usage.
- **Note:** the JSON data endpoint `GET /vets` (E17) has **no** template and is **not** consumed by
  any view — `vets/vetList.html` renders the server-side `listVets` model, not the `/vets` JSON.

### API usage (how the frontend reaches the backend)

The frontend communicates with the backend **entirely through server-rendered navigation and HTML
form submissions** — no client-side XHR/`fetch`/AJAX. — confidence **high** (searched
`fetch(`/`axios`/`xhr`/`ajax`/`JSON` in templates → only the DOM-timer inline script matches).

| UI action | HTTP call (backend endpoint) | Mechanism | Evidence |
|-----------|------------------------------|-----------|----------|
| Owner search | `GET /owners` (E6) | `<form th:action="@{/owners}" method="get">` | `owners/findOwners.html:9` |
| Create owner | `POST /owners/new` (E4) | `<form th:object="${owner}" method="post">` | `owners/createOrUpdateOwnerForm.html:8` |
| Edit owner | `POST /owners/{ownerId}/edit` (E9) | same form, edit route | `createOrUpdateOwnerForm.html:8`; `OwnerController` |
| Add / edit pet | `POST /owners/{ownerId}/pets/**` (E11/E13) | `<form th:object="${pet}" method="post">` | `pets/createOrUpdatePetForm.html:11` |
| Add visit | `POST /owners/{ownerId}/pets/{petId}/visits/new` (E15) | `<form th:object="${visit}" method="post">` | `pets/createOrUpdateVisitForm.html:30` |
| Navigate (home/find/vets/error) | `GET /`, `/owners/find`, `/vets.html`, `/oups` | `th:href` anchors in the navbar | `fragments/layout.html:41-60` |
| Vet-list pagination | `GET /vets.html?page=N` (E16) | `th:href` anchors | `vets/vetList.html:30-52` |

### Auth / session

**No authentication context reaches the frontend.** — confidence **high**.

The project declares no `spring-boot-starter-security` and no security config (api-inventory
"Auth note" and G3); every route is served unauthenticated, and no template reads a user/token/role.
The only session-bound frontend behaviour is **locale**: a `SessionLocaleResolver` +
`LocaleChangeInterceptor` honour a `?lang=` query parameter, and messages are resolved server-side
via `#{...}` keys.

- **Evidence:** `system/WebConfiguration.java` (locale resolver/interceptor, per capability-baseline
  C5); `messages/messages_*.properties` (11 locales); absence of Spring Security in `pom.xml`
  (api-inventory G3). No cookie/token handoff, no `Authorization` header logic in templates.

### CSS approach

**SCSS compiled to a single plain CSS file, themed on top of Bootstrap; no CSS-in-JS, no Tailwind,
no CSS modules.** — confidence **high**.

- **Author-written SCSS** in `src/main/scss/`: `petclinic.scss` (entry; `@import "bootstrap"` then
  Spring-branded variable overrides — `$spring-green: #6db33f`, `$body-bg`, `$border-radius-base: 0`,
  etc.), which `@import`s `typography.scss`, `header.scss`, `responsive.scss`
  (`petclinic.scss:14,212-214`).
- **Compiled output** `static/resources/css/petclinic.css` is a single committed stylesheet that
  bundles Bootstrap's compiled output (including Bootstrap 5's `--bs-*` CSS custom properties in
  `:root`) plus the PetClinic theme. Font faces (`montserratregular`, `varela_roundregular`) are
  declared in `typography.scss` and served from `static/resources/fonts/`.
- **Vendor CSS** (Font Awesome `font-awesome.min.css`) is linked separately from WebJars
  (`layout.html:13`).
- **Confidence:** high — SCSS sources, compiled CSS, and the `@import`/variable structure are all
  read directly.

### Capabilities mapped

The presentation tier maps to capability-baseline **C5 — Clinic Web Presentation & Localization**
(the layout shell, SCSS/CSS, WebJar assets, i18n). Individual screens surface the domain
capabilities: owner views → **C1**, pet views → **C2**, visit view → **C3**, vet list → **C4**, home
→ **C5**, error demo/`error.html` → **C5/C7**. No presentation asset is scoped to the technical
capabilities C6 (persistence) or C7 (ops) beyond the `error.html` fallback. — confidence **high**
(cross-referenced to capability-baseline Sections 1–3).

---

## Frontend asset ownership

**Layout-scoped / globally shared across all capabilities** — assets are **not** capability-scoped,
tenant-scoped, or per-surface. — confidence **high**.

- A **single** layout shell (`fragments/layout.html`) is `th:replace`d by **all 9** page templates
  (verified: `welcome`, `findOwners`, `ownersList`, `ownerDetails`, `createOrUpdateOwnerForm`,
  `createOrUpdatePetForm`, `createOrUpdateVisitForm`, `vetList`, `error`).
- A **single** stylesheet (`petclinic.css`) and a **single** vendor JS bundle
  (`bootstrap.bundle.min.js`) are loaded for every page by that shared layout.
- **Shared-asset coupling (observed):** any change to the layout shell, `petclinic.css`, or the
  Bootstrap/Font-Awesome WebJar version affects **every** screen and **every** capability (C1–C5)
  simultaneously; there is no per-capability CSS/JS isolation.
- **Evidence:** `layout.html:11-14,84`; `find … -name '*.css'` → one file; `grep -l fragments/layout
  templates/**` → all page templates.

## Frontend deployment boundary

**Embedded within the server-rendered runtime packaging (bundled with the backend deployment).** —
confidence **high**. Not independently deployable, not CDN-served, not statically hosted separately.

- Templates live under `src/main/resources/templates/` and static assets under
  `src/main/resources/static/`, so both are packaged **inside the Spring Boot fat JAR** and served by
  the same JVM/process that serves the backend routes. There is no separate frontend artifact, no CDN
  configuration, and no static-host deploy step.
- The Kubernetes manifest (`k8s/petclinic.yml`, per capability-baseline C7) deploys the **single**
  `petclinic` container; the UI ships and versions with it.
- **Evidence:** repo layout (`src/main/resources/{templates,static}`); `pom.xml`
  `spring-boot-maven-plugin` (fat-jar packaging); absence of any CDN/asset-host config (searched).

## Routing model

- **Route ownership:** server-side — Spring MVC `@Controller` classes own every route (E1–E18).
- **Routing mechanism:** annotation-based request mapping (`@GetMapping`/`@PostMapping`) resolving
  Thymeleaf view names / redirects; navigation is full-page loads and form submits.
- **Centralized vs distributed:** **centralized** at the framework/dispatcher-servlet level, though
  the mappings themselves are distributed across 5 controllers by domain package (`owner`, `vet`,
  `system`).
- **Server-side vs client-side:** **entirely server-side** — no client-side router config exists.
- **Route→capability mapping:** as tabulated under Routes / screens (C1–C5).
- **Confidence:** high. **Evidence:** api-inventory Part 1 (E1–E18); the five controller classes;
  navbar `th:href` links in `layout.html:41-60`.

## UI surfaces vs frontend platform

**Single UI surface** — a unified clinic web UI — riding on a **single shared frontend platform**.
No separate administrative UI, customer portal, embedded widget, reporting dashboard, or scheduler
surface is observed. — confidence **high**.

- **The one surface:** the clinic web application (home, owner/pet/visit management, vet directory,
  error demo). All screens share one navbar, one layout, one audience, and one URL space; there is no
  namespace, entry-asset, template tree, or build output that separates a second surface.
- **The shared frontend platform:** the layout shell (`fragments/layout.html`), the compiled
  `petclinic.css`, the WebJar vendor bundles (Bootstrap 5.3.8, Font Awesome 4.7.0), the fonts, and
  the `?lang=` i18n mechanism — all global.
- **Cross-surface relationship:** **not applicable** — with a single surface there is nothing to
  isolate or couple. Were the surface list to grow, the current structure is **tightly shared** (one
  layout, one CSS, one bundle, one deploy artifact) rather than partially isolated or independently
  owned.
- **Evidence:** the 12-template tree (no second app shell/entry point); single layout used by all
  pages; single CSS/JS asset set.

## Frontend asset loading strategy

**Template-driven inclusion of globally-shared assets, with one page-scoped inline script.** —
confidence **high**. This is a **mixed but simple** model, dominated by template-driven global loading.

- **Template-driven inclusion (global):** the shared layout injects, in `<head>`, the Font-Awesome
  vendor CSS then `petclinic.css`; at end-of-`<body>` it injects the Bootstrap JS bundle. Load order
  is fixed by the layout for every page. Evidence: `layout.html:13-14` (CSS), `:84` (JS).
- **Globally-bundled vendor assets:** Bootstrap/Font-Awesome loaded on **every** page via
  `@{/webjars/...}` (webjars-locator resolves the path). Evidence: `layout.html:13,84`.
- **Page-scoped assets:** exactly one — the inline `hideMessages()` script on `ownerDetails.html`
  (`:80-91`). No page loads a dedicated external JS/CSS file of its own.
- **Not present (searched):** lazy-loaded / dynamic `import()` chunks, runtime-injected/remote
  scripts, and CDN-based absolute-URL loading — **none observed**. All assets are same-origin,
  served by the app.

## Frontend state management

**No explicit frontend state management pattern detected.** — confidence **high**.

No Redux/Vuex/NgRx, no global `window` state object, no client-side store, and no event bus exist
(no author-written JS at all). State is **server-session-bound**: the server renders each view from
the request model, and locale is held in the HTTP session (`SessionLocaleResolver`). There is **no
client-side hydration and no embedded-JSON handoff** — templates bind server model attributes
directly with `th:text`/`th:field`/`th:each` (e.g. `vetList.html` iterates `${listVets}`;
`ownerDetails.html` renders `${owner}`), not into an inline JSON blob. The single inline script
performs a DOM-only timer with no state.

- **Evidence:** `grep JSON/json_encode/JSON.stringify/window./localStorage` in `templates/**` →
  only the DOM-timer script; `system/WebConfiguration.java` (session locale); `vets/vetList.html`,
  `owners/ownerDetails.html` (direct server-model binding).

## Design system & component catalogue

There is **no standalone component library, no design-system package, and no Storybook**. Reuse is
achieved with **server-side Thymeleaf fragments** and **SCSS variables**, not a JS component
catalogue. Findings from the Step 5.6 scan:

- **Shared component library:** **none detected.** Directories `components/`, `ui/`, `shared/`,
  `lib/`, `design-system/`, `ui-kit/`, `.storybook/`, and package names containing
  `design-system`/`ui-kit`/`component-lib(rary)` were searched repo-wide — none exist. There is no
  `package.json`, so no workspace/npm component package can exist.
- **Reusable components observed (Thymeleaf fragments — server-side):**

  | Fragment | File path | Scope | Cross-use evidence (import paths) | Confidence |
  |----------|-----------|-------|-----------------------------------|------------|
  | `layout` + `menuItem` | `templates/fragments/layout.html` | **globally shared** (app shell) | `th:replace`d by all 9 page templates | high |
  | `input` (form field) | `templates/fragments/inputField.html` | **globally shared** (across C1/C2/C3 forms) | `createOrUpdateOwnerForm.html:10-14`, `createOrUpdatePetForm.html`, `createOrUpdateVisitForm.html` | high |
  | `select` (form select) | `templates/fragments/selectField.html` | **domain-specific** (pet form only) | `createOrUpdatePetForm.html` | high |

- **Design tokens:** **no dedicated token file detected** (`tokens.*`, `design-tokens.*`, `theme.*`,
  `variables.css`, `_variables.scss`, `tokens.json` — none found). The closest artefacts are
  **SCSS variables** in `src/main/scss/petclinic.scss` (colour tokens `$spring-green #6db33f`,
  `$spring-dark-green`, `$spring-brown`, `$spring-grey`, `$spring-light-grey`; semantic assignments
  `$body-bg`/`$text-color`/`$link-color`; radius `$border-radius-base: 0`) — categories observed:
  **colour, semantic colour role, border-radius**. The `--bs-*` CSS custom properties present in the
  compiled `petclinic.css` `:root` are **Bootstrap 5's own generated variables**, not a hand-authored
  token layer. Confidence **high**; **medium** on calling the SCSS variables "tokens" (they are a
  Bootstrap-theme override set, not an abstracted token system).
- **Storybook / component docs:** **not observed** (`.storybook/`, `*.stories.*` → none).
- **Shared vs domain-specific split:** the app shell and form-input fragment are shared platform
  concerns; the select fragment is the only domain-scoped fragment. All are **server-side templates**,
  not packaged components. — confidence **high**.
- **Packaging:** **not observed** — no npm package, no monorepo workspace; fragments are co-located
  in the application's `templates/` tree and ship inside the Spring Boot JAR. — confidence **high**.

## UI architecture / evolution signals

Observed frontend architectural characteristics and evolution constraints (non-prescriptive —
pressure described, no solutions):

- **Rendering ownership is server-bound.** Every screen is produced by a Spring MVC controller +
  Thymeleaf template in-process; the UI cannot render independently of the backend runtime. This is a
  **frontend/backend runtime coupling**.
- **No independently deployable UI boundary was observed.** Templates and static assets are packaged
  and versioned inside the single `petclinic` JAR/container; a UI change requires rebuilding and
  redeploying the whole application.
- **Frontend assets are globally shared across all capabilities.** One layout shell, one stylesheet,
  and one vendor bundle serve C1–C5, so any shared-asset change is a **cross-capability release
  coordination** point.
- **Routing is centralized server-side.** There is no client-side route ownership per feature; UI
  navigation and view resolution are a backend concern.
- **UI composition is server-side fragment inclusion**, not client-side component composition —
  reuse is coupled to the Thymeleaf/Spring rendering model.
- **Server-session-bound rendering (locale) and direct server-model binding** create a dependency on
  backend rendering for all dynamic content; there is no client state layer to evolve independently.
- **The CSS toolchain is opt-in and out of CI**, and the compiled CSS is a committed artifact — a
  latent **drift risk** between `src/main/scss/**` and the shipped `petclinic.css` if the `-P css`
  step is skipped.
- **Cross-surface relationships: not applicable** — a single surface exists today, so there is no
  observed cross-surface coupling or isolation to evolve.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The shipped `static/resources/css/petclinic.css` is a faithful compile of `src/main/scss/**` (no manual drift). | The `css` profile is opt-in and not run in CI (`mvnw -B verify` / `gradlew build`), yet `petclinic.css` is committed. | The rendered UI could differ from the SCSS source of truth; SCSS edits may not reach production. | Run `./mvnw package -P css` and diff the regenerated `petclinic.css` against the committed file. |
| A2 | Bootstrap/Font-Awesome are resolved from WebJars at build/runtime (no committed vendor JS/CSS). | No vendor bundle files exist under `static/`; only `@{/webjars/...}` links and `pom.xml` WebJar deps. | Asset-loading conclusions (global vendor bundle) would be wrong if assets are served from elsewhere. | Inspect the built JAR's `/webjars/**` classpath and the running app's asset requests. |
| A3 | The SCSS `$spring-*` variables are the design "tokens"; there is no separate abstracted token system. | Only SCSS variables + Bootstrap `--bs-*` generated vars were found; no `tokens.*`/`theme.*` files. | A token layer elsewhere (e.g. an external theme) would be missed. | Confirm with the UI owner that theming is limited to `petclinic.scss`. |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Does the CAKE `Scheduling MFE` / patient-portal (React micro-frontend) model represent planned future scope for this platform, or an unrelated catalog for a different product? | Determines whether an entire client-side/MFE presentation tier is future scope vs out-of-scope noise for this repo. | Not implemented in code; recorded as Future/Intended State (Not Implemented). | Product / architecture owner |
| Q2 | Who (if anyone) consumes the `GET /vets` JSON endpoint, given no in-repo template renders it? | Determines whether an external/JS client of the UI exists beyond the server-rendered surface. | Unknown — no in-repo consumer (api-inventory G1); no template uses it. | Platform / API owner |
| Q3 | Is the CSS `-P css` compilation expected to run in CI/release, or is regenerating `petclinic.css` a manual developer step? | Affects SCSS→CSS drift risk (A1) and reproducibility of the shipped UI. | Currently manual/opt-in; CI does not invoke it. | Platform / build owner |

