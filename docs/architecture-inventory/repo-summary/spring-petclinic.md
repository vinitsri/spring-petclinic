# Repo Summary — `spring-petclinic`

- **Repository:** `spring-petclinic` (also known as: Spring PetClinic, PetClinic, Petclinic platform application)
- **Location:** Repository: `spring-petclinic`, path: `.`
- **Deployable name:** `petclinic` (Kubernetes `Service`/`Deployment` name in `k8s/petclinic.yml`; Maven `<name>petclinic</name>`; Maven artifact `spring-petclinic`)
- **Git URL / base ref:** `https://github.com/vinitsri/spring-petclinic` @ `feature/VE-216/aidlc`
- **Build coordinates:** `org.springframework.samples:spring-petclinic:4.0.0-SNAPSHOT`, Spring Boot `4.1.0`, Java 17

## README vs repository (Grounding §3)

- **README claims (docs pass):** Spring Boot app buildable with Maven **or** Gradle; runs on
  Java 17+; in-memory H2 by default with MySQL/PostgreSQL profiles; H2 console at
  `/h2-console`; no `Dockerfile` (use `spring-boot:build-image` buildpacks); CSS compiled from
  SCSS via the Maven `css` profile; test `main()` apps for fast IDE feedback
  (`README.md`).
- **Repository shows (tree/manifest pass):** all README claims are corroborated by
  `pom.xml`, `build.gradle`, `application*.properties`, `src/main/scss/`, and
  `db/{h2,mysql,postgres}/`. Additionally present but **not mentioned in README** (disk-only):
  Kubernetes manifests `k8s/petclinic.yml` + `k8s/db.yml`; the `deploy-and-test-cluster.yml`
  GitHub workflow (kind cluster); GraalVM native build (`native-maven-plugin`,
  `PetClinicRuntimeHints.java`); CycloneDX SBOM plugin; 11-locale i18n bundles
  (`src/main/resources/messages/`); Caffeine cache backing.
- **Stale-doc marker:** README's legacy-slides link is self-flagged as pre–Spring Boot and not
  reflective of the current implementation (README lines note this). **Marked Stale doc.**
- **Unknown (neither pass proved):** the build/publish origin of the `dsyer/petclinic` image
  referenced by `k8s/petclinic.yml` — no Dockerfile or image-publish pipeline in the clone.
- **Source-of-truth note:** the CAKE catalog's `Scheduler Service` (appointment booking,
  tenant isolation, `/v1/...` endpoints, MFE identity headers) is **not** present in this
  repository's code, endpoints, messaging, or frontend. Per the Source-of-Truth rule, code
  wins: recorded as *Future/Intended State (Not Implemented)* / not applicable to this repo.

## 1. Primary purpose

Sample **veterinary clinic management** web application: register/manage pet owners, track
their pets and pet types, list veterinarians and their specialties, and record visits. This
matches the CAKE domain narrative for the PetClinic application.
- **Evidence:** `README.md`; `src/main/java/org/springframework/samples/petclinic/owner/`,
  `.../vet/`; `src/main/resources/templates/`.

## 2. Main runtime/service type

Single-deployable **Spring Boot 4.1.0** server-rendered web application — Spring MVC
controllers with Thymeleaf views — on Java 17. Not a monorepo; one Spring module.
- **Evidence:** `pom.xml` (`spring-boot-starter-webmvc`, `-thymeleaf`, parent `4.1.0`,
  `<java.version>17`); `build.gradle`; `PetClinicApplication.java` (`@SpringBootApplication`).

## 3. Key entrypoints

- Application main: `src/main/java/org/springframework/samples/petclinic/PetClinicApplication.java`
  (`SpringApplication.run`, `@ImportRuntimeHints(PetClinicRuntimeHints.class)`).
- HTTP entrypoint: container port `8080` (`k8s/petclinic.yml`), local `http://localhost:8080/`
  (`README.md`).
- Start commands: `./mvnw spring-boot:run`, `./gradlew bootRun` (`README.md`).
- Test/dev `main()` apps: `PetClinicIntegrationTests`, `MysqlTestApplication`,
  `PostgresIntegrationTests` (`src/test/java/...`).

## 4. Important modules / packages

Under `org.springframework.samples.petclinic`:
- **`owner`** — `Owner`, `Pet`, `PetType`, `Visit`, `OwnerController`, `PetController`,
  `VisitController`, `OwnerRepository`, `PetTypeRepository`, `PetValidator`, `PetTypeFormatter`.
- **`vet`** — `Vet`, `Specialty`, `Vets`, `VetController`, `VetRepository`.
- **`model`** — `BaseEntity`, `NamedEntity`, `Person` (shared JPA superclasses).
- **`system`** — `WelcomeController`, `CrashController`, `CacheConfiguration`, `WebConfiguration`.
- **Evidence:** `src/main/java/org/springframework/samples/petclinic/**`.

## 5. External integrations

No third-party service integrations. Outbound dependencies are the relational database (JDBC)
and browser-delivered WebJars assets.
- **Evidence:** `pom.xml` / `build.gradle` (`postgresql`, `mysql-connector-j`, `h2`,
  `webjars.npm:bootstrap`, `font-awesome`, `webjars-locator-lite`); no HTTP/SDK client deps.

## 6. Data stores / state handling

- **Mechanism:** Spring Data JPA (`JpaRepository`) over Hibernate ORM; snake-case physical
  naming; `spring.jpa.hibernate.ddl-auto=none`; `open-in-view=false`; batch fetch size 16.
- **Databases:** H2 in-memory (default), MySQL, PostgreSQL — selected via Spring profile.
- **Migration tool:** **None** (no Flyway/Liquibase/Alembic). Schema and seed data are applied
  by `spring.sql.init` from `db/${database}/schema.sql` and `data.sql`.
- **Tables (primary domain):** `owners`, `pets`, `types`, `visits`, `vets`, `specialties`,
  `vet_specialties`.
- **Foreign keys / coupling points (single schema, single service):** `pets.owner_id → owners`,
  `pets.type_id → types`, `visits.pet_id → pets`, `vet_specialties.vet_id → vets`,
  `vet_specialties.specialty_id → specialties`. These are intra-service relations within one
  schema — **not cross-domain/cross-service coupling** (no other service owns these tables).
- **Notable constraint:** unique pet name per owner — `unique_owner_pet_name` as
  `CREATE UNIQUE INDEX ... ON pets (owner_id, LOWER(name))` (PostgreSQL) and
  `UNIQUE (owner_id, name)` on `VARCHAR_IGNORECASE` (H2); enforced in-app via
  `DataIntegrityViolationException` handling keyed on the `unique_owner_pet_name` name.
- **Evidence:** `src/main/resources/application.properties`, `application-postgres.properties`,
  `application-mysql.properties`; `src/main/resources/db/postgres/schema.sql`, `db/h2/schema.sql`;
  `src/main/java/.../owner/OwnerRepository.java`, `.../owner/PetController.java` (lines ~128,
  ~170, `isDuplicatePetNameViolation`).

## 7. Messaging / async / event mechanisms

**None.** No SNS, SQS, Kafka, RabbitMQ, or EventBridge dependencies, no topic/queue names, and
no publisher/consumer code. Caching (`@EnableCaching`, JCache/Caffeine on the `vets` cache) is
present but is not a messaging mechanism.
- **Evidence:** absence in `pom.xml`/`build.gradle`; `src/main/java/.../system/CacheConfiguration.java`.

## 8. APIs exposed or consumed

**Exposed** (Spring MVC, mostly HTML views):
- `GET /` and `GET /welcome` — welcome page (`WelcomeController`).
- `GET /owners/new`, `POST /owners/new`, `GET /owners/find`, `GET /owners`,
  `GET /owners/{ownerId}`, `GET|POST /owners/{ownerId}/edit` (`OwnerController`).
- `GET /pets/new`, `POST /pets/new`, `GET|POST /pets/{petId}/edit` (`PetController`);
  visit endpoints (`VisitController`).
- `GET /vets.html` (paginated HTML) and `GET /vets` (JSON, `@ResponseBody Vets`) (`VetController`).
- `GET /oups` — deliberate exception demo (`CrashController`).
- `GET /actuator/*` — all Actuator endpoints exposed; `GET /h2-console`; k8s probes
  `/livez`, `/readyz`.
- **Consumed:** none (no outbound API clients).
- **Evidence:** controllers under `src/main/java/.../owner/`, `.../vet/`, `.../system/`;
  `application.properties` (`management.endpoints.web.exposure.include=*`); `k8s/petclinic.yml`.

## 9. Deployment / runtime clues

- **Local DB:** `docker-compose.yml` services `mysql` (`mysql:9.7`) and `postgres`
  (`postgres:18.4`), consumed automatically via `spring-boot-docker-compose`.
- **Kubernetes:** `k8s/petclinic.yml` (`Service` type `NodePort` :80→8080, `Deployment` image
  `dsyer/petclinic`, `SPRING_PROFILES_ACTIVE=postgres`, service-binding volume, liveness
  `/livez` / readiness `/readyz`); `k8s/db.yml` (PostgreSQL `Deployment` + `Secret` of type
  `servicebinding.io/postgresql`).
- **Container build:** no `Dockerfile`; buildpacks via `spring-boot:build-image` (`README.md`).
- **CI:** `.github/workflows/maven-build.yml` (`./mvnw -B verify`, JDK 17),
  `gradle-build.yml` (`./gradlew build`), `deploy-and-test-cluster.yml` (kind + `kubectl apply -f k8s/`).
- **Dev envs:** `.devcontainer/`, `.gitpod.yml`, Codespaces badge.
- **Native:** GraalVM native image support (`native-maven-plugin`, `build.gradle` graalvm
  plugin, `PetClinicRuntimeHints.java`).
- **Evidence:** files as cited above.

## 10. Security / auth clues

**No authentication or authorization** — no Spring Security (or OAuth/JWT) dependency. The app
is unauthenticated. Actuator is fully exposed (`management.endpoints.web.exposure.include=*`,
commented as dev/test-only) and the H2 console is enabled. Bean Validation constraints exist on
domain input (e.g. `Owner.telephone` `@Pattern("\\d{10}")`, `@NotBlank`) but these are input
validation, not access control.
- **Evidence:** no security artifact in `pom.xml`/`build.gradle`; `application.properties`;
  `README.md` (h2-console); `src/main/java/.../owner/Owner.java`.

## 11. Observability / logging / tracing

- **Actuator:** `spring-boot-starter-actuator` provides health/metrics/info; all endpoints
  exposed for monitoring.
- **Cache metrics:** JCache statistics enabled and exposed via JMX (`CacheConfiguration`);
  Caffeine backs the `vets` cache.
- **Logging:** SLF4J/Logback (Spring Boot default); `logging.level.org.springframework=INFO`.
- **Not present:** no Micrometer registry (e.g. Prometheus), no distributed tracing
  (Micrometer Tracing/OpenTelemetry/Zipkin) dependency. **Needs validation** if distributed
  tracing is expected.
- **Evidence:** `pom.xml`; `src/main/java/.../system/CacheConfiguration.java`;
  `application.properties`.

## 12. Architecture-decision files / feature flags

- **Decision records:** no `docs/`, ADR, or design-decision files existed in the repo before
  this inventory. Architectural governance is expressed through build enforcement:
  Checkstyle + `nohttp-checkstyle`, `spring-javaformat`, Maven Enforcer, and CycloneDX SBOM.
- **Feature-flag system:** **None detected** — no LaunchDarkly, Unleash, Split, Flagsmith, or
  custom flag store; no flag keys referenced in domain code. The only profile-based toggling is
  Spring profiles selecting the database (`mysql`/`postgres`).
- **Evidence:** repository tree (absence of `docs/`/ADRs); `pom.xml`/`build.gradle` plugin
  blocks; `application-*.properties`.

## 13. Open questions / ambiguities

- Canonical build tool (Maven vs Gradle) for release is unstated.
- Provenance of the `dsyer/petclinic` deploy image is not in-repo.
- Production restriction of open Actuator endpoints and the H2 console is unverified.
- Whether distributed tracing/metrics export is an expected but missing capability.
- (All mirrored in the ABQ section below.)

## 14. Frontend stack

Server-side rendered UI — **no client-side JS framework**. Thymeleaf templates
(`src/main/resources/templates/**/*.html`, including `fragments/layout.html`) styled with
Bootstrap `5.3.8` and font-awesome `4.7.0` delivered via WebJars; site styling authored in SCSS
(`src/main/scss/petclinic.scss`, `header.scss`, `typography.scss`, `responsive.scss`) and
compiled to `src/main/resources/static/resources/css/petclinic.css` by the `libsass-maven-plugin`
(Maven `css` profile).
- **Build tooling:** libsass via Maven (no webpack/vite; no `package.json`).
- **MFE / federation:** **none** — no Module Federation, `single-spa`, `customElements.define`,
  or `__webpack_share_scopes__`; no JS bundler.
- **Checked directories:** `src/main/resources/static/`, `static/resources/css`,
  `static/resources/images`, `static/resources/fonts`, `src/main/scss/`,
  `src/main/resources/templates/`. No `.js`/`.ts`/`.vue`/`.jsx` sources and no
  `public/`, `web/`, `wwwroot/`, `resources/js/`, or `assets/js/` directories exist.
- **i18n:** `SessionLocaleResolver` + `LocaleChangeInterceptor` (`?lang=`) with 11 locale
  bundles (`messages_{de,en,es,fa,hi,ja,ko,pt,ru,tr}.properties` + default).
- **Evidence:** `src/main/resources/templates/`, `src/main/scss/`, `pom.xml`
  (`libsass-maven-plugin`, WebJars), `src/main/java/.../system/WebConfiguration.java`,
  `src/main/resources/messages/`.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The default runtime datastore is in-memory H2 unless a Spring profile overrides it. | `application.properties` sets `database=h2`; MySQL/PostgreSQL live behind profiles. | Data-persistence and migration analysis in later stages could target the wrong engine. | Confirm the profile used per environment (`SPRING_PROFILES_ACTIVE`); `k8s/petclinic.yml` uses `postgres`. |
| A2 | Open Actuator/H2 console exposure is intentional for dev only. | Inline comments in `application.properties` say "only for development and testing". | A production deployment would expose management/DB surfaces. | Verify production config/ingress controls with the platform owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | Container image `dsyer/petclinic` build/publish pipeline is not in this repository. | Reproducible deployment and provenance for `k8s/petclinic.yml`. | Release/DevOps owner | Locate or add the image build (buildpacks) and publish pipeline; document the registry. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Is Maven or Gradle the canonical build/release path? | Both are wired into CI; downstream SBOM/native automation must pick one. | — | Platform build owner |
| Q2 | Should distributed tracing/metrics export (Micrometer/OTel) be added? | No tracing/metrics-registry dependency exists today. | Not present; add only if observability requirements demand it. | Platform/observability owner |
| Q3 | Does the CAKE `Scheduler Service` (appointment booking) represent planned scope for this app? | It appears in the catalog but has no code realization here. | Treat as future/intended or unrelated to this repo until implemented. | Product / architecture owner |
