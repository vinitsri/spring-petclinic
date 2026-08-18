# Component Internals — `system-web`

- **Component:** `system-web`
- **Repo / path:** `repo` → `src/main/java/org/springframework/samples/petclinic/system`
- **Stage:** Architecture discovery — component-internals working-model (batch 2 of 2)
- **Grounding:** Derived by reading the component's Java source (four `@Controller`/
  `@Configuration` classes plus `package-info`), the Thymeleaf templates and message bundles it
  drives, and the `application.properties` / `pom.xml` keys that wire its behaviour. Every
  load-bearing claim cites a `path:symbol`, a template, a message key, a config key, or a POM
  coordinate. Claims the code cannot settle are marked **`Unverifiable — Missing Source
  Evidence`**.
- **Maintenance contract:** This model is authoritative and reused by downstream phases. Any
  later change to this component's internals MUST update this file in the same change so the
  model never drifts from the code.

> **Evidence legend:** `code` = a Java symbol/annotation · `tmpl` = a Thymeleaf template ·
> `msg` = a `messages/*.properties` key · `config` = `application.properties` · `pom` =
> `pom.xml` dependency. A behaviour that raises an error is **loud**; one that is
> dropped/ignored without an error is **silent**.

---

## 1. Purpose & boundary

`system-web` is the **cross-cutting platform / presentation-shell layer** of Spring PetClinic.
Unlike the domain components (`owner-management`, `vet-management`) it owns **no business entity
and no database table**. Instead it owns the four framework-integration seams that every domain
page depends on but no domain package should re-implement:

1. **Web presentation shell & entry points** — the home page (`GET /`) and the shared HTML
   layout that all domain views render into (`WelcomeController.java`, `templates/welcome.html`,
   `templates/fragments/layout.html`).
2. **Internationalization (i18n)** — request-scoped locale selection and message resolution
   across 11 locales, switched by a `?lang=` query parameter (`WebConfiguration.java`,
   `messages/messages*.properties`, `config: spring.messages.basename`).
3. **Caching infrastructure** — enables Spring's cache abstraction and programmatically defines
   the single application cache, `vets`, consumed by `vet-management`
   (`CacheConfiguration.java`).
4. **Error / diagnostics surface** — a deliberate exception-throwing endpoint used to
   demonstrate the global error page, plus the error view contract itself
   (`CrashController.java`, `templates/error.html`).

**It is responsible for:**

- Serving the application root view (`WelcomeController.welcome()` → `welcome` view).
- Configuring the `LocaleResolver` and `LocaleChangeInterceptor` that every controller and
  template rely on for language selection (`WebConfiguration.java`).
- Declaring the `vets` cache and turning on `@EnableCaching` for the whole application
  (`CacheConfiguration.java`); the *use* of that cache lives in `vet-management`.
- Providing the `/oups` diagnostic endpoint and the `error` view that Spring Boot's default
  error handling resolves to (`CrashController.java`, `templates/error.html`).
- The shared navigation/layout fragment and the i18n message catalogue that all domain screens
  reuse (`templates/fragments/layout.html`, `messages/`).

**It is explicitly NOT responsible for:**

- Any domain data or persistence — it defines no `@Entity`, no repository, and touches no
  schema/seed SQL. (Owner/Pet/Visit belong to `owner-management`; Vet/Specialty to
  `vet-management`.)
- Populating the `vets` cache — it only *creates* the cache; `VetController`/`VetRepository`
  read/evict it (`vet-management`).
- Security, authentication, or authorization — there is no Spring Security dependency or config
  in this component (`pom` has no `spring-boot-starter-security`).
- The Actuator/H2/observability endpoints themselves — those are framework beans toggled by
  `application.properties` (`management.endpoints.web.exposure.include=*`), not Java owned by
  this package. This component only *co-locates* the operational config surface; see §5 / ABQ.
- Static assets, WebJars, and CSS bundling — served by Spring Boot's resource handling and the
  build, not by code in this package.

---

## 2. Core concepts (exhaustive)

The concepts this component defines or owns:

| # | Concept | One-line role | Primary evidence |
|---|---------|---------------|------------------|
| 2.1 | **Welcome / home view** | The application root page (`GET /`). | `WelcomeController.java:25`, `templates/welcome.html` |
| 2.2 | **Shared layout fragment** | The single HTML shell (nav bar, footer, head) every view composes into. | `templates/fragments/layout.html` |
| 2.3 | **Locale resolver** | Where the "current language" for a request is stored (HTTP session). | `WebConfiguration.localeResolver():33` |
| 2.4 | **Locale change interceptor** | The mechanism that reads `?lang=` and switches the stored locale. | `WebConfiguration.localeChangeInterceptor():45` |
| 2.5 | **Interceptor registration** | Wires 2.4 into the MVC request pipeline. | `WebConfiguration.addInterceptors():56` |
| 2.6 | **Message catalogue (i18n bundles)** | The 11-locale key→text catalogue resolved by `#{...}` in templates and by validation. | `messages/messages*.properties`, `config: spring.messages.basename` |
| 2.7 | **`vets` cache definition** | The named JCache created at startup and consumed by vet-management. | `CacheConfiguration.petclinicCacheConfigurationCustomizer():36` |
| 2.8 | **Cache abstraction enablement** | `@EnableCaching` — turns `@Cacheable`/`@CacheEvict` anywhere in the app into live behaviour. | `CacheConfiguration.java:32` |
| 2.9 | **Cache statistics toggle** | JMX statistics on the `vets` cache. | `CacheConfiguration.cacheConfiguration():50` |
| 2.10 | **Error-demo endpoint** | `GET /oups` — deliberately throws to exercise the error page. | `CrashController.triggerException():32` |
| 2.11 | **Global error view** | The `error` view Spring Boot resolves to on any unhandled exception / error status. | `templates/error.html` |
| 2.12 | **View / template resolution contract** | How a returned view name (`"welcome"`, `"error"`) becomes rendered HTML. | `config: spring.thymeleaf.mode`, `templates/` |

---

## 3. Per-concept model

### 2.1 Welcome / home view

- **Definition.** The landing page served at the application root.
- **Representation & storage.** A single `@GetMapping("/")` handler returning the logical view
  name `"welcome"` (`WelcomeController.java:25-28`); the view is `templates/welcome.html`. No
  model attributes are added — the page is static apart from the i18n `#{welcome}` title
  (`tmpl: welcome.html`).
- **Lifecycle.** Created implicitly by Spring MVC component-scan of the `@Controller`; served on
  every `GET /`. No persistence, no mutation, no versioning.
- **Invariants & enforcement.** The returned string MUST match a resolvable view
  (`templates/welcome.html`). If the template were missing, view resolution fails **loudly**
  (`500` → the error view). The class is package-private (`class WelcomeController`) — it is not
  an API extension point.
- **Extension procedure.** To change the home page, edit `templates/welcome.html` (and add any
  new label to **all 11** `messages/messages*.properties` files, per 2.6). To add model data,
  inject a service and add attributes in `welcome()`.
- **Failure modes.** A view name typo returns a non-existent view → framework raises
  `500` (loud). Adding a `#{key}` with no matching message key renders the literal default text
  or the key itself (Thymeleaf), a **silent** presentation defect.

### 2.2 Shared layout fragment

- **Definition.** The one HTML shell (`<head>`, top navigation, Spring footer logo, Bootstrap
  bundle) that every page — welcome, error, and all domain views — is composed into via
  Thymeleaf's `th:replace="~{fragments/layout :: layout (~{::body},'<menu>')}"`.
- **Representation & storage.** `templates/fragments/layout.html`, fragment
  `layout (template, menu)`. It hard-codes the four nav items and their target routes: `/` (home),
  `/owners/find`, `/vets.html`, and `/oups` (`tmpl: layout.html`, `menuItem` replacements). Each
  nav label is an i18n key (`#{home}`, `#{findOwners}`, `#{vets}`, `#{error}`).
- **Lifecycle.** Referenced at render time by each page's root `<html th:replace=...>`; the
  `menu` argument drives the `active` CSS class on the matching nav item.
- **Invariants & enforcement.** The `menu` selector string passed by a page (e.g. `'home'`,
  `'error'`, `'owners'`, `'vets'`) must match the per-item `active==menu` comparison to
  highlight the current tab. A mismatch simply leaves nothing highlighted — **silent**.
- **Extension procedure.** To add a global nav entry, add a `menuItem` `th:replace` block in
  `layout.html` with a route, a Font-Awesome glyph, and a `#{key}`; add that key to all 11
  message bundles. To change global chrome (favicon, CSS, JS), edit the `<head>`/`<script>`
  blocks here — it propagates to every page.
- **Failure modes.** Because this is the single shell, a broken fragment (bad Thymeleaf
  expression) breaks **every** page, not one — a wide blast radius. This is the component's
  highest-leverage file.

### 2.3 Locale resolver

- **Definition.** The strategy that determines and remembers the "current locale" for a request.
- **Representation & storage.** A `SessionLocaleResolver` bean named `localeResolver`, with
  default `Locale.ENGLISH` (`WebConfiguration.localeResolver():33-37`). The chosen locale is
  stored **in the HTTP session** — so it persists across requests for the same session and is
  **not** shared across sessions or derived from the `Accept-Language` header.
- **Lifecycle.** Instantiated once as a singleton bean at context startup; read on every request
  by the MVC dispatcher and by `#{...}` message lookups; mutated by the interceptor (2.4). Reset
  to English implicitly when a new session starts (no prior stored locale).
- **Invariants & enforcement.** The bean **must** be named `localeResolver` — Spring MVC looks
  up the `LocaleResolver` by that conventional bean name. Rename it and MVC silently falls back
  to its default `AcceptHeaderLocaleResolver`, changing behaviour **silently**
  (`code: @Bean localeResolver`).
- **Extension procedure.** To change default language, change `resolver.setDefaultLocale(...)`.
  To switch storage strategy (e.g. cookie-based persistence), replace `SessionLocaleResolver`
  with `CookieLocaleResolver` in this same bean method — no other code changes needed.
- **Failure modes.** Because storage is session-scoped, an anonymous/first request always renders
  in English regardless of browser language — expected, not a bug. Losing the session (new
  browser, cleared cookies) resets the language to English **silently**.

### 2.4 Locale change interceptor

- **Definition.** The request interceptor that lets a user switch language via a URL parameter.
- **Representation & storage.** A `LocaleChangeInterceptor` bean whose param name is set to
  `"lang"` (`WebConfiguration.localeChangeInterceptor():45-49`). On any request carrying
  `?lang=<code>` it calls the resolver (2.3) to store that locale.
- **Lifecycle.** Runs `preHandle` on every request that reaches the MVC pipeline (after
  registration, 2.5). When `?lang=` is present it mutates the session locale; otherwise it is a
  no-op and the existing/default locale stands.
- **Invariants & enforcement.** The param name is `lang` (matches `?lang=de`, `?lang=es`, …).
  The interceptor does **not** validate the code against the available bundles — an unknown code
  like `?lang=zz` is accepted and stored; message lookups then fall back to the default bundle
  (`messages.properties`) **silently** rather than erroring.
- **Extension procedure.** To rename the query parameter, change `setParamName(...)`. To reject
  unsupported locales you would add a `setSupportedLocales(...)`/custom guard (not present today
  — see ABQ).
- **Failure modes.** An invalid or unsupported `lang` value silently degrades to the default
  language (no error). A locale with no bundle file resolves each missing key to its default
  inline text.

### 2.5 Interceptor registration

- **Definition.** The wiring that inserts the locale interceptor (2.4) into the MVC pipeline.
- **Representation & storage.** `WebConfiguration implements WebMvcConfigurer` and overrides
  `addInterceptors(registry)` to `registry.addInterceptor(localeChangeInterceptor())`
  (`WebConfiguration.java:56-58`).
- **Lifecycle.** Invoked once by Spring MVC during context initialization to assemble the
  interceptor chain.
- **Invariants & enforcement.** If this override were removed, the `LocaleChangeInterceptor`
  bean would still exist but would **never run** — `?lang=` would be silently ignored. The
  registration is the load-bearing link, not the bean definition alone.
- **Extension procedure.** Add further `registry.addInterceptor(...)` calls here for any new
  cross-cutting interceptor. `@SuppressWarnings("unused")` on the class signals the beans are
  framework-invoked, not called directly.
- **Failure modes.** Forgetting to register a new interceptor is a **silent** no-op — the bean
  compiles and exists but has no effect.

### 2.6 Message catalogue (i18n bundles)

- **Definition.** The key→localized-text catalogue that backs every `#{...}` in templates and
  every validation message code.
- **Representation & storage.** Property files under `messages/`: a default
  `messages.properties` plus 10 locale variants — `de, en, es, fa, hi, ja, ko, pt, ru, tr`
  (**11 files total**). The base name is wired by `spring.messages.basename=messages/messages`
  (`config`). Keys include page labels (`welcome`, `home`, `vets`), form labels, pagination
  (`first/next/previous/last`), and error strings (`error.404`, `error.500`, `error.general`,
  `somethingHappened`) (`msg: messages.properties`).
- **Lifecycle.** Loaded by Spring's `MessageSource` at startup; resolved per request against the
  locale from 2.3. Adding a locale = add a `messages_<code>.properties` file (no code change).
  Adding a key = add it to the default file (and ideally every locale file).
- **Invariants & enforcement.** Resolution falls back **default-ward**: `messages_<locale>` →
  `messages.properties` → the inline default text in the template, or the raw key. A key present
  in the default file but **missing** from a locale file renders the default-language text for
  that one label — a **silent**, per-key degradation, never an error.
- **Extension procedure.** To add a user-facing string: add the key to `messages.properties`
  first (the fallback), then translate it into each `messages_<code>.properties`; reference it
  in the template as `th:text="#{yourKey}"`. Omitting the default entry means every locale falls
  through to the raw key text.
- **Failure modes.** Untranslated keys → mixed-language pages (silent). A malformed properties
  line can fail bundle loading for that locale (loud at startup for that resource).

### 2.7 `vets` cache definition

- **Definition.** The single named cache the application maintains, holding the veterinarian
  listing so repeated `/vets` requests avoid the database.
- **Representation & storage.** Created programmatically by a `JCacheManagerCustomizer` bean:
  `cm.createCache("vets", cacheConfiguration())` (`CacheConfiguration.java:36-38`). It is a
  JCache (JSR-107) cache; the provider is **Caffeine** via `spring-boot-starter-cache` +
  `javax.cache:cache-api` + `com.github.ben-manes.caffeine:caffeine` (`pom.xml:48,68-69,82-83`).
  The cache lives **in-process, in JVM heap** — there is no distributed/remote cache.
- **Lifecycle.** The cache is created once at startup by the customizer. Entries are written and
  evicted **by `vet-management`** (`@Cacheable("vets")` / `@CacheEvict` on `VetRepository` —
  outside this component). This component never reads or writes cache entries; it only
  provisions the cache.
- **Invariants & enforcement.** The cache **name must be exactly `"vets"`** — it is the contract
  string that `vet-management` references by literal. A rename here without the matching change
  in the vet package makes `@Cacheable("vets")` reference a non-existent cache; depending on
  cache-resolver config that fails **loudly** at first access or silently no-ops. Size/TTL are
  **not** set here (the JCache `MutableConfiguration` exposes no size limit); the code comment at
  `CacheConfiguration.java:45-47` explicitly notes real limits must be set through the provider
  (Caffeine) mechanism — none is configured, so eviction is provider-default. **Unverifiable —
  Missing Source Evidence** for any explicit max-size/TTL: none exists in this component.
- **Extension procedure.** To add a second cache, add another `cm.createCache("<name>", …)` call
  in the customizer (or a new customizer bean) and reference `<name>` from the consuming
  `@Cacheable`. To bound the `vets` cache, add a Caffeine spec (e.g. `spring.cache.caffeine.spec`
  in config, or a provider-specific configuration) — nothing bounds it today.
- **Failure modes.** Because no size/TTL is set, the cache grows to hold the vets listing
  unbounded in principle (in practice one small list). Stale data is prevented only by the
  `@CacheEvict` in the vet package — a change there, not here, governs freshness.

### 2.8 Cache abstraction enablement

- **Definition.** The switch that makes Spring's caching annotations active application-wide.
- **Representation & storage.** `@EnableCaching` on `CacheConfiguration`
  (`CacheConfiguration.java:32`), with `@Configuration(proxyBeanMethods = false)`.
- **Lifecycle.** Processed once at context startup; installs the caching AOP advisor so
  `@Cacheable`/`@CacheEvict` anywhere become live.
- **Invariants & enforcement.** Removing `@EnableCaching` makes every `@Cacheable` in the app a
  **silent** no-op — methods still run, nothing is cached, no error. This is a global toggle
  owned here.
- **Extension procedure.** Nothing to add for new caches at this level; `@EnableCaching` covers
  all of them. Keep it present.
- **Failure modes.** Its absence degrades performance silently (every vet lookup hits the DB) —
  no functional break, easy to miss.

### 2.9 Cache statistics toggle

- **Definition.** JMX-exposed hit/miss statistics for the `vets` cache.
- **Representation & storage.** `new MutableConfiguration<>().setStatisticsEnabled(true)`
  (`CacheConfiguration.cacheConfiguration():50`). The class comment states statistics "become
  accessible via JMX" (`CacheConfiguration.java:27-29`).
- **Lifecycle.** Applied when the cache is created; statistics accumulate for the JVM lifetime.
- **Invariants & enforcement.** Purely observational; disabling it removes JMX cache metrics but
  changes no functional behaviour.
- **Extension procedure.** Add `.setManagementEnabled(true)` or further JCache options on the
  same `MutableConfiguration` if richer JMX management is needed.
- **Failure modes.** None functional; only a monitoring gap if disabled.

### 2.10 Error-demo endpoint

- **Definition.** A diagnostic route that deliberately throws to demonstrate global error
  handling — reachable from the nav bar's "Error" item.
- **Representation & storage.** `@GetMapping("/oups")` on `CrashController` throws a
  `RuntimeException` unconditionally (`CrashController.java:31-35`). It is linked from
  `layout.html`'s fourth nav item (`/oups`, `#{error}`).
- **Lifecycle.** Invoked on demand; never returns normally — the exception propagates to Spring
  Boot's default error handling, which forwards to `/error` and resolves the `error` view (2.11).
- **Invariants & enforcement.** By design it always fails **loudly** (that *is* its function).
  The class is package-private and has no side effects, persistence, or model.
- **Extension procedure.** This is demo-only; a production build could remove `CrashController`
  and its `/oups` nav link. To simulate other failures, throw a different exception type here.
- **Failure modes.** None to guard — failure is the intended output. Note it is **unauthenticated
  and always enabled** (no profile guard), so it is publicly reachable in any environment.

### 2.11 Global error view

- **Definition.** The page rendered for any unhandled exception or error HTTP status.
- **Representation & storage.** `templates/error.html`, composed into the shared layout with
  menu `'error'`. It switches on `${status}`: `404`→`#{error.404}`, `500`→`#{error.500}`,
  else→`#{error.general}`, and prints `${message}` and `#{somethingHappened}` (`tmpl:
  error.html`; `msg: error.404/error.500/error.general/somethingHappened`).
- **Lifecycle.** Resolved by Spring Boot's default error machinery (`BasicErrorController` →
  `/error` → view named `error`) — the framework supplies `status`/`message` model attributes;
  this component supplies only the *view*. No Java in this package handles `/error` directly.
- **Invariants & enforcement.** The view name `error` and the availability of the `status`/
  `message` attributes are a **framework contract**, not code owned here. Renaming the template
  breaks the mapping **silently** (Boot falls back to Whitelabel error page).
- **Extension procedure.** Edit `error.html` to add status cases or change copy; add any new
  `error.*` message key to all bundles. To customize error *behaviour* (not just the view) you
  would add an `@ControllerAdvice`/`ErrorController` — none exists today.
- **Failure modes.** A missing/renamed template silently reverts to the Whitelabel page; an
  unhandled status not in the switch renders the `error.general` branch.

### 2.12 View / template resolution contract

- **Definition.** How a controller's returned string becomes rendered HTML.
- **Representation & storage.** Spring Boot's Thymeleaf auto-configuration maps a view name to
  `templates/<name>.html`; `spring.thymeleaf.mode=HTML` (`config`) selects HTML parsing.
  Controllers here return bare names (`"welcome"`) and rely entirely on this convention.
- **Lifecycle.** Applied per request during view rendering.
- **Invariants & enforcement.** A returned name with no matching `templates/<name>.html` fails
  **loudly** (`500`). Static resource caching is governed by
  `spring.web.resources.cache.cachecontrol.max-age=12h` (`config`).
- **Extension procedure.** Add a new view by adding `templates/<name>.html` and returning
  `"<name>"` from a handler.
- **Failure modes.** View-name/template mismatch → 500 (loud), which itself renders 2.11.

---

## 4. Primary control flows

**A. Home-page request (`GET /`).**
1. Request enters the MVC `DispatcherServlet`.
2. `LocaleChangeInterceptor.preHandle` (registered at `WebConfiguration.addInterceptors():56`)
   inspects for `?lang=`; if present it stores that locale via `SessionLocaleResolver`
   (`WebConfiguration.localeResolver():33`), else the session/default (English) locale stands.
3. Handler `WelcomeController.welcome()` (`:25`) returns `"welcome"`.
4. Thymeleaf resolves `templates/welcome.html`, which `th:replace`s `fragments/layout ::
   layout` (2.2); `#{...}` keys resolve against `messages/messages_<locale>.properties` with
   default-ward fallback (2.6).
5. Rendered HTML returns; static assets carry a 12h cache header (`config:
   spring.web.resources.cache.cachecontrol.max-age`).

**B. Language switch (`GET /<any>?lang=de`).**
1. Same entry; `LocaleChangeInterceptor` finds `lang=de`, calls the resolver to persist
   `Locale.GERMAN` into the **session** (2.4 → 2.3).
2. The originally-targeted handler proceeds; every subsequent request in that session renders in
   German until changed again — no validation of the code occurs (an unknown code degrades
   silently to default, 2.4/2.6).

**C. Error demonstration (`GET /oups`) and the global error path.**
1. `CrashController.triggerException()` (`:32`) throws `RuntimeException` unconditionally.
2. The exception is not caught by any `@ControllerAdvice` in this app; Spring Boot's default
   error handling forwards to `/error` and populates `status`/`message`.
3. Boot resolves the view named `error` → `templates/error.html` (2.11), which composes the
   shared layout and switches on `${status}` to pick `#{error.404|500|general}`.
4. The same path renders for *any* unhandled exception or error status anywhere in the app — not
   just `/oups`.

**D. `vets` cache provisioning (startup, not per-request).**
1. At context startup `@EnableCaching` (`CacheConfiguration.java:32`) installs the caching
   advisor.
2. Boot detects the JCache/Caffeine provider (`pom.xml:48,68-69,82-83`) and invokes the
   `JCacheManagerCustomizer` bean (`:36`), which calls `cm.createCache("vets", …)` with
   statistics enabled (`:50`).
3. Thereafter, `vet-management`'s `@Cacheable("vets")` populates and `@CacheEvict` clears this
   cache — those reads/writes happen **outside** this component; here only the cache's existence
   is guaranteed.

---

## 5. Persistence & schema evolution

- **This component has no datastore.** It defines no entity, no repository, no table, and
  references no `schema.sql`/`data.sql`. There is nothing to migrate or seed within its scope.
- The only "persistence-like" state it owns is the **in-JVM `vets` cache** (2.7) — heap-resident,
  not durable, rebuilt on every restart, with no configured size/TTL bound in this component.
- Related config it *reads/co-locates* (defined in `application.properties`, not Java here):
  `spring.messages.basename=messages/messages` (i18n base), `spring.thymeleaf.mode=HTML`,
  `spring.web.resources.cache.cachecontrol.max-age=12h`, and the operational surface
  `management.endpoints.web.exposure.include=*` (all Actuator endpoints exposed — flagged
  dev-only by inline comment, a production exposure concern; owned by config, not this package).
- "Schema evolution" for this component means **adding message keys / locale files** (2.6) and
  **editing templates** (2.1/2.2/2.11) — there is no DB migration framework in play.

---

## 6. Surface → internals map

| Surface (public) | Kind | Internal mechanism it drives |
|------------------|------|------------------------------|
| `GET /` | Route (read-only) | `WelcomeController.welcome()` → `welcome` view via shared layout (2.1/2.2). No side effects. |
| `GET /oups` | Route (side-effect: throws) | `CrashController.triggerException()` throws → Boot error path → `error` view (2.10/2.11). |
| `?lang=<code>` (query param on any route) | Cross-cutting mutator | `LocaleChangeInterceptor` writes the session locale via `SessionLocaleResolver` (2.4/2.3). **Mutates session state.** |
| `error` view (framework-invoked `/error`) | Rendered contract | `templates/error.html` status-switch + i18n (2.11). Read-only render. |
| `vets` cache (SPI, not HTTP) | Startup-provisioned resource | `CacheConfiguration` `createCache("vets")` (2.7); consumed by vet-management's `@Cacheable`. |
| `#{...}` message lookups (used by all views/validation) | i18n resolution | `MessageSource` over `messages/messages*.properties` keyed by current locale (2.6). |
| Shared nav/layout (`fragments/layout :: layout`) | Template SPI | `layout.html` shell every page composes into (2.2). |
| `@EnableCaching` | App-wide toggle | Activates `@Cacheable`/`@CacheEvict` everywhere (2.8). Absence = silent no-op. |

Read-only surfaces: `GET /`, the `error` render, `#{...}` lookups, the layout fragment.
Mutating surfaces: `?lang=` (session locale). Side-effecting: `GET /oups` (throws by design).

---

## 7. Change / extension guide

- **To add a user-facing string:** add the key to `messages/messages.properties` **first** (it
  is the fallback), then to each `messages_<code>.properties`; reference via `th:text="#{key}"`.
  *Silent rejection:* omit the default entry and every locale falls through to the raw key text —
  no error (2.6).
- **To add a supported language:** drop a `messages_<code>.properties` file next to the others —
  no code change; the base name `messages/messages` picks it up automatically (2.6). *Silent
  gap:* keys you forget to translate render in the default language.
- **To restrict which `?lang=` values are honoured:** you must add validation (e.g.
  `setSupportedLocales(...)` or a guard) in `WebConfiguration.localeChangeInterceptor()` — today
  any code is accepted and unknown codes silently fall back to default (2.4).
- **To change the default language or locale storage:** edit the `localeResolver()` bean
  (`WebConfiguration.java:33`). Keep the bean name `localeResolver` or MVC silently ignores it
  (2.3).
- **To add a global nav item or change site chrome:** edit `templates/fragments/layout.html`
  (2.2) and add its `#{key}` to all 11 bundles. This file is the highest-blast-radius change in
  the component — it renders on every page.
- **To add or bound a cache:** add another `cm.createCache("<name>", …)` in
  `CacheConfiguration.petclinicCacheConfigurationCustomizer()` and reference `<name>` from the
  consumer's `@Cacheable`. To bound `vets`, add a Caffeine size/TTL spec — none exists today, so
  the cache is provider-default-unbounded (2.7). *Silent trap:* a name mismatch between
  `createCache` and `@Cacheable` yields a missing cache.
- **To customize error handling (not just the page):** add a `@ControllerAdvice`/
  `ErrorController` — none exists; today all errors flow through Boot defaults into `error.html`
  (2.11). Renaming `error.html` silently reverts to the Whitelabel page.
- **To harden the operational surface:** the fully-open Actuator exposure lives in
  `application.properties` (`management.endpoints.web.exposure.include=*`) — change it there, not
  in this package.

---

## 8. Assumptions, Blockers & Open Questions (ABQ)

| ID | Type | Statement | Basis / Evidence | Resolution owner |
|----|------|-----------|------------------|------------------|
| A1 | Assumption | The `error` view name and `status`/`message` model attributes are supplied by Spring Boot's default `BasicErrorController`; no code in this package handles `/error`. | `templates/error.html` uses `${status}`/`${message}`; no `ErrorController` in `system/`. | Framework contract — verify on Boot 4.1.0 upgrade. |
| A2 | Assumption | The `vets` cache provider is Caffeine-backed JCache. | `pom.xml:48,68-69,82-83` (`spring-boot-starter-cache`, `javax.cache:cache-api`, `caffeine`); `CacheConfiguration` uses only the generic JCache API. | Verify at runtime / on dependency change. |
| Q1 | Open question | What eviction/size/TTL bounds the `vets` cache? None is set in this component. | `CacheConfiguration.cacheConfiguration():49-50` sets only statistics; comment at `:45-47` says real limits need the provider mechanism. **Unverifiable — Missing Source Evidence** here. | Platform owner / vet-management. |
| Q2 | Open question | Is `?lang=` restricted to the 11 shipped locales in any environment? | `LocaleChangeInterceptor` (`:45`) sets no supported-locale list; unknown codes degrade silently. | Product / platform owner. |
| Q3 | Open question | Is `GET /oups` (always-on, unauthenticated) removed or gated in production builds? | `CrashController.java` has no profile guard; nav link is unconditional in `layout.html`. | Platform / security owner. |
| Q4 | Open question | Are the fully-exposed Actuator endpoints and H2 console network-restricted in production? | `application.properties` (`management.endpoints.web.exposure.include=*`, inline dev-only comment). Config surface co-located with this component but not Java-owned. | Security / platform owner. |
