# Component Internals — `vet-management`

- **Component:** `vet-management`
- **Repo / path:** `repo` → `src/main/java/org/springframework/samples/petclinic/vet`
- **Stage:** Architecture discovery — component-internals working-model (batch 1 of 2)
- **Grounding:** Derived by reading the component's Java source, JPA mappings, JAXB annotations,
  the `VetRepository` contract, the `system/CacheConfiguration` bean it depends on, and the
  DDL/seed SQL for `vets`/`specialties`/`vet_specialties`. Every load-bearing claim cites a
  `path:symbol` or a schema/seed artifact. Claims the code cannot settle are marked
  **`Unverifiable — Missing Source Evidence`**.
- **Maintenance contract:** Authoritative and reused downstream. Any later change to this
  component's internals MUST update this file in the same change.

> **Evidence legend:** `code` = Java symbol/annotation · `schema` = DDL file · `seed` = `data.sql`
> · `config` = properties/config. A behaviour that raises an error is **loud**; one dropped/ignored
> without error is **silent**.

---

## 1. Purpose & boundary

`vet-management` is the **veterinarian directory** of Spring PetClinic. It owns the **Vet** entity,
the **Specialty** reference vocabulary, and the **many-to-many** association between them, and it
exposes them **read-only** in two shapes: a paginated HTML list and a machine-readable (JSON/XML)
resource list.

**It is responsible for:**

- The **Vet** read model — listing vets with their specialties (`Vet.java`, `VetController.java`).
- The **Specialty** reference vocabulary (`Specialty.java`) and the **`vet_specialties`** join.
- The **`Vets`** marshalling wrapper used to serialise the vet collection to XML/JSON
  (`Vets.java`).
- The read-only persistence contract for `vets`, `specialties`, and `vet_specialties`
  (`src/main/resources/db/{h2,mysql,postgres}/schema.sql`).
- Participating in the **`vets` cache** (`@Cacheable("vets")` on `VetRepository`, cache created by
  `system/CacheConfiguration`).

**It is explicitly NOT responsible for:**

- **Owners / pets / visits / pet-types** — owned by `owner-management` (`../owner`). This component
  never reads or writes those tables.
- **Creating, updating, or deleting vets or specialties** — there is **no write path at runtime**.
  `VetRepository` exposes only `findAll` overloads; both `vets` and `specialties` (and their join
  rows) are **seed-only** (see §3 and §5).
- **Owning the cache infrastructure** — the `vets` cache is *declared* here (`@Cacheable`) but
  *configured* in `system/CacheConfiguration` (outside this package); this component only depends on
  it.
- **Authentication / authorization / multi-tenancy** — none; both endpoints are anonymous and global.
- **Scheduling / appointments** — no appointment or availability concept exists in this component
  (contrary to the unrelated scheduling model the CAKE tenant catalog describes; see §8).

---

## 2. Core concepts (exhaustive)

1. **Vet** — a veterinarian; the read-only aggregate of this component.
2. **Specialty** — the reference vocabulary of veterinary specialties (radiology, surgery,
   dentistry).
3. **Vet↔Specialty association** — the many-to-many link via the `vet_specialties` join table.
4. **Read-only repository contract** — `VetRepository extends Repository` (not `JpaRepository`);
   query surface deliberately limited to `findAll`.
5. **The `vets` cache** — `@Cacheable("vets")` + JCache configuration.
6. **`Vets` marshalling wrapper & content negotiation** — how the collection serialises to XML/JSON.
7. **Specialty projection & ordering** — `getSpecialties()` sorted view + `getNrOfSpecialties()`.
8. **Pagination** — page size 5 for the HTML list.
9. **Entity identity & `isNew()`** — shared `BaseEntity` identity semantics.
10. **Person identity fields** — `firstName`/`lastName` via the shared `model.Person` superclass.

---

## 3. Per concept

### 3.1 Vet

- **Definition** — A veterinarian: a person (`firstName`/`lastName`) with a set of `Specialty`s.
  The read-only aggregate root of this component.
- **Representation & storage** — `Vet.java:43-45` `@Entity @Table(name = "vets")` extending
  `model.Person`. The only local field is `specialties` (`Vet.java:47-50`). Table `vets`
  (`schema.sql`): `id` identity PK, `first_name`, `last_name` (`VARCHAR(30)` in H2/MySQL, `TEXT` in
  Postgres), plus index `vets_last_name`. Note there is **no `specialties` column** on `vets`; the
  association lives entirely in the join table (§3.3).
- **Lifecycle** —
  - *Create / Update / Delete:* **none at runtime.** `VetRepository` exposes only reads
    (`VetRepository.java:38-57`), so vets are introduced solely by `data.sql`
    (`db/h2/data.sql:1-6` seeds six vets).
  - *Read:* `VetRepository.findAll()` (all vets) and `findAll(Pageable)` (paged), both cached and
    read-only (`VetRepository.java:44-56`).
  - *Actor:* seed script / DBA for existence; `VetController` for reads.
- **Invariants & enforcement** — `firstName`/`lastName` carry `@NotBlank @Size(max=30)` from
  `Person.java:32-38`. **Because there is no bind/create path, these annotations are never exercised
  at runtime** in this component — they constrain hypothetical writes only. The DB columns are the
  effective shape for seeded data.
- **Extension procedure** — Adding a vet at runtime is **not supported by code**: add an
  `INSERT INTO vets …` to all three `data.sql`, or introduce a write repository method (see §3.4)
  and a controller/form (none exists today). Any new vet field needs the `@Column` on `Vet.java`
  plus the column in all three `schema.sql`; with `ddl-auto=none` a missing column fails **loudly**.
- **Failure modes** — None at runtime for reads. A structural mismatch between `Vet` mapping and the
  `vets` DDL fails loudly at query time.

### 3.2 Specialty

- **Definition** — A veterinary specialty (radiology, surgery, dentistry); a reference vocabulary,
  read-only in this component.
- **Representation & storage** — `Specialty.java:28-30` `@Entity @Table(name = "specialties")`
  extending `NamedEntity` (just `id` + `name`). Table `specialties`: `id` identity PK,
  `name VARCHAR(80)`, index `specialties_name` (`schema.sql`). Seeded by `data.sql`
  (`db/h2/data.sql:8-10`).
- **Lifecycle** —
  - *Create / Update / Delete:* **none at runtime.** There is no `SpecialtyRepository`; specialties
    are seed-only. New specialties enter only via `INSERT INTO specialties …` in `data.sql`.
  - *Read:* only **transitively**, through `Vet.getSpecialties()` when a vet is loaded (the
    `@ManyToMany(EAGER)` fetch, §3.3). There is no standalone specialty listing.
  - *Actor:* seed script for existence; loaded as part of the Vet aggregate.
- **Invariants & enforcement** — `name` is `@NotBlank` (via `NamedEntity.java:33-35`); never
  exercised at runtime (no write path). No uniqueness constraint on `specialties.name` in any schema.
- **Extension procedure** — Add `INSERT INTO specialties VALUES (default, '<name>');` to all three
  `data.sql`, then link it to vets via `vet_specialties` rows (§3.3). There is no admin surface.
- **Failure modes** — A `vet_specialties` row referencing a non-existent `specialty_id` violates
  `fk_vet_specialties_specialties` **loudly** at seed time (`db/h2/schema.sql:28`).

### 3.3 Vet↔Specialty association (`vet_specialties`)

- **Definition** — A vet may hold many specialties and a specialty may be held by many vets: a
  many-to-many association.
- **Representation & storage** — `Vet.java:47-50`:
  `@ManyToMany(fetch = FetchType.EAGER) @JoinTable(name = "vet_specialties",
  joinColumns=@JoinColumn(vet_id), inverseJoinColumns=@JoinColumn(specialty_id))`. Join table
  `vet_specialties` (`schema.sql`): `vet_id`, `specialty_id`, FKs to `vets`/`specialties`
  (`db/h2/schema.sql:23-28`). MySQL and Postgres additionally declare `UNIQUE (vet_id, specialty_id)`
  (`db/mysql/schema.sql:19`, `db/postgres/schema.sql:17`); **H2 does not** — so on the default H2
  profile a duplicate pairing is not constraint-prevented (relevant only if a write path is added).
  Seed links: `db/h2/data.sql:12-16`.
- **Lifecycle** —
  - *Create/Delete of links:* seed-only. `Vet.addSpecialty(Specialty)` (`Vet.java:70-72`) exists as
    an in-memory mutator but is **never called by any runtime path** in this component (no write
    controller/repository) — it is used by tests/aggregate construction only.
  - *Read:* eager — loading a vet pulls its specialties in the same unit of work. With
    `spring.jpa.open-in-view=false`, EAGER is what makes `getSpecialties()` safe to call while
    rendering.
- **Invariants & enforcement** — Referential integrity via the two FKs (loud at seed time). The
  join is realised through `getSpecialtiesInternal()` which lazily instantiates a `HashSet` if the
  field is null (`Vet.java:52-57`), so `getNrOfSpecialties()`/`getSpecialties()` never NPE on a
  vet with no specialties.
- **Extension procedure** — To link a vet to a specialty in seed data, add
  `INSERT INTO vet_specialties VALUES (<vet_id>, <specialty_id>);` to all three `data.sql`. To
  enable runtime linking you would need a write repository method + controller (absent today).
- **Failure modes** — Dangling FK in seed data fails loudly. On H2 (no unique constraint) duplicate
  seed pairs would silently create redundant join rows, inflating `getNrOfSpecialties()`.

### 3.4 Read-only repository contract

- **Definition** — The deliberate design that vets can only be *read*.
- **Representation & storage** — `VetRepository extends
  org.springframework.data.repository.Repository<Vet, Integer>` (`VetRepository.java:38`) — the
  **minimal** Spring Data base interface, which exposes **no methods by default** (unlike
  `JpaRepository`, used by `owner-management`). The interface declares exactly two methods:
  `Collection<Vet> findAll()` and `Page<Vet> findAll(Pageable)`, both
  `@Transactional(readOnly = true) @Cacheable("vets")` (`VetRepository.java:44-56`).
- **Lifecycle / enforcement** — Because no `save`/`delete` is declared, there is **no compiled write
  surface**; attempts to mutate vets do not exist to call. `readOnly = true` also hints the tx
  manager to skip dirty-checking/flush.
- **Extension procedure** — To allow writes you must **widen the interface** (e.g. add `save`, or
  extend `JpaRepository`/`CrudRepository`) — a deliberate architectural change, and you must then
  evict/disable the `vets` cache on writes (§3.5) or reads will go stale.
- **Failure modes** — None at runtime. The narrow contract is itself the guardrail against
  accidental vet mutation.

### 3.5 The `vets` cache

- **Definition** — A JCache-backed cache named `vets` that memoises the result of the `findAll`
  queries.
- **Representation & storage** — `@Cacheable("vets")` on both `findAll` overloads
  (`VetRepository.java:45,55`). The cache is **created and configured outside this component** in
  `system/CacheConfiguration` (`@Configuration @EnableCaching`, bean
  `petclinicCacheConfigurationCustomizer` calls `cm.createCache("vets", …)` with statistics enabled,
  `system/CacheConfiguration.java:36-51`). Caching is enabled process-wide by `@EnableCaching` there.
- **Lifecycle / enforcement** — First `findAll` populates the cache under a key derived from the
  method args (no-arg vs `Pageable`); subsequent identical calls are served from cache. **There is
  no `@CacheEvict` anywhere** — consistent with the read-only model: since vets never change at
  runtime, the cache is never invalidated. Cache statistics are exposed via JMX (JCache).
- **Extension procedure** — If a vet write path is ever added, it **must** annotate the mutator with
  `@CacheEvict("vets")` (or `allEntries = true`) or the list endpoints will serve stale data
  silently. If a second cached read method is added, be aware both overloads currently share the
  same cache name `vets`.
- **Failure modes** — Stale reads are impossible today (data is immutable at runtime). The risk is
  entirely latent, activated only by a future write path.

### 3.6 `Vets` marshalling wrapper & content negotiation

- **Definition** — A thin wrapper object that lets the vet collection serialise cleanly to XML (and
  JSON) for the machine-readable endpoint.
- **Representation & storage** — `Vets` (`Vets.java:30-43`): `@XmlRootElement` with a lazily
  initialised `List<Vet>` exposed via `@XmlElement getVetList()`. `Vet.getSpecialties()` is annotated
  `@XmlElement` (`Vet.java:59-60`) so specialties nest in the marshalled output. It is **not** a JPA
  entity — purely a serialisation DTO.
- **Lifecycle / enforcement** — Built per request in `VetController.showResourcesVetList`
  (`VetController.java:65-72`): `new Vets()`, then `getVetList().addAll(vetRepository.findAll())`,
  returned as `@ResponseBody`. Spring content negotiation picks JSON or XML (JAXB) based on the
  request `Accept`/extension; the JAXB annotations drive the XML shape.
- **Extension procedure** — To change the serialised shape, adjust the `@XmlElement`/`@XmlRootElement`
  annotations on `Vets`/`Vet`. Adding fields to `Vet` automatically flows into JSON; for XML they
  need appropriate JAXB annotations.
- **Failure modes** — None functional; a missing JAXB annotation would just omit/rename an XML
  element (silent shape change), not error.

### 3.7 Specialty projection & ordering

- **Definition** — The stable, name-sorted view of a vet's specialties and their count.
- **Representation & storage** — `Vet.getSpecialties()` streams the internal `Set`, sorts by
  `NamedEntity::getName`, and returns a `List` (`Vet.java:59-64`). `getNrOfSpecialties()` returns the
  set size (`Vet.java:66-68`). Backing store is the unordered `HashSet` from
  `getSpecialtiesInternal()`.
- **Lifecycle / enforcement** — Sorting happens on every read to give deterministic display/marshal
  order regardless of the `Set`'s iteration order. Used by the HTML template and the XML/JSON output.
- **Extension procedure** — To change ordering, edit the `Comparator` in `getSpecialties()`. Callers
  that need the count should use `getNrOfSpecialties()` rather than materialising the list.
- **Failure modes** — None; a vet with a null specialties field is handled by the lazy-init in
  `getSpecialtiesInternal()`.

### 3.8 Pagination

- **Definition** — The HTML vet list is paged, 5 vets per page.
- **Representation & storage** — `VetController.findPaginated` builds `PageRequest.of(page-1, 5)`
  and calls `vetRepository.findAll(pageable)` (`VetController.java:59-63`); `addPaginationModel`
  publishes `currentPage`, `totalPages`, `totalItems`, `listVets` to the model
  (`VetController.java:50-57`).
- **Lifecycle / enforcement** — `GET /vets.html?page=N` (default 1). The **`/vets`
  (`@ResponseBody`) endpoint is NOT paged** — it returns *all* vets via the no-arg `findAll`
  (§3.6).
- **Extension procedure** — Change the literal `5` (`VetController.java:60`) to adjust page size.
- **Failure modes** — `page` < 1 → negative page index in `PageRequest.of` → `IllegalArgumentException`
  (loud).

### 3.9 Entity identity & `isNew()`

- **Definition** — Shared identity semantics; see `owner-management` §3.9. Vet and Specialty both
  inherit `BaseEntity` (`id`, `isNew()`), with DB-generated `IDENTITY` ids
  (`model/BaseEntity.java:33-49`). Because this component has no write path, `isNew()` is not
  branched on at runtime here (unlike in `owner-management`).

### 3.10 Person identity fields

- **Definition** — `Vet` reuses `model.Person`'s `firstName`/`lastName` (`Person.java:31-39`,
  `@NotBlank @Size(max=30)`), mapped to `vets.first_name`/`last_name`. This is the **same shared
  superclass** used by `Owner`; changes to `Person` affect both components. `vets.last_name` is
  indexed (`vets_last_name`) though no last-name query exists in this component today.

---

## 4. Primary control flows

Both flows are **read-only**; there is no mutation path in this component.

**A. HTML vet list** — `GET /vets.html?page=N` → `VetController.showVetList`
(`VetController.java:44-48`) → `findPaginated(page)` → `vetRepository.findAll(PageRequest.of(page-1,
5))` (cached, read-only) → `addPaginationModel` publishes `listVets` + paging attributes → renders
`vets/vetList` (`templates/vets/vetList.html`). Each vet's specialties render via the sorted
`getSpecialties()` projection (§3.7).

**B. Machine-readable vet resource** — `GET /vets` → `VetController.showResourcesVetList`
(`VetController.java:65-72`) → `new Vets()` + `getVetList().addAll(vetRepository.findAll())` (no-arg,
cached) → returned `@ResponseBody`. Content negotiation serialises to **JSON** (Jackson) or **XML**
(JAXB via `@XmlRootElement`/`@XmlElement` on `Vets`/`Vet`) per the request. This endpoint is
**unpaged** — it returns the full vet collection.

Both flows read through the same `@Cacheable("vets")` methods, so after the first call of each shape
the collection is served from the `vets` cache (§3.5) until process restart (no eviction exists).

---

## 5. Persistence & schema evolution

- **Datastore.** Same relational store as the rest of PetClinic — H2 in-memory by default
  (`application.properties:1`), MySQL/Postgres by switching the `database` property. Schema/seed
  applied by `spring.sql.init` (not Hibernate; `ddl-auto=none`), with **no migration framework**
  (no Flyway/Liquibase).
- **Tables owned by this component:** `vets`, `specialties`, and the join `vet_specialties` (FKs
  `fk_vet_specialties_vets`, `fk_vet_specialties_specialties`). Defined in all three
  `db/{h2,mysql,postgres}/schema.sql`.
- **How schema changes are applied.** Hand-edit the DDL in **all three** dialect files (there are no
  versioned migrations). H2 drops-and-recreates the tables each boot
  (`db/h2/schema.sql:1-3` drops `vet_specialties`, `vets`, `specialties` in FK-safe order);
  MySQL/Postgres use `CREATE TABLE IF NOT EXISTS`.
- **Seeding & idempotency.** `data.sql` seeds six vets, three specialties, and five vet/specialty
  links via plain `INSERT` with **no upsert** (`db/h2/data.sql:1-16`) — **not idempotent** on a
  persistent (non-H2) store across restarts. On H2 the drop/recreate makes re-seeding clean.
- **Dialect divergence that matters:** the `vet_specialties` **`UNIQUE (vet_id, specialty_id)`**
  constraint exists in MySQL/Postgres but **not** in H2 (§3.3) — a latent difference that only
  matters if a write path is introduced.

---

## 6. Surface → internals map

| Surface (HTTP) | Handler | Internal mechanism driven | Kind |
|---|---|---|---|
| `GET /vets.html?page=N` | `VetController.showVetList` | `VetRepository.findAll(Pageable)` (cached) → paginated `vetList` view | read-only |
| `GET /vets` | `VetController.showResourcesVetList` | `VetRepository.findAll()` (cached) → `Vets` wrapper → JSON/XML | read-only |
| *(no endpoint)* | — | **Vet create/update/delete** | **absent** — no write repository (§3.4) |
| *(no endpoint)* | — | **Specialty CRUD** | **absent** — seed-only (§3.2) |
| *(no endpoint)* | — | **vet↔specialty link management** | **absent** — seed-only (§3.3) |

Internal (non-HTTP) surfaces: `VetRepository` (Spring Data `Repository`, read-only, cached, §3.4);
the `vets` JCache defined by `system/CacheConfiguration` (dependency, §3.5); the `Vets` JAXB wrapper
(§3.6); shared `model.Person`/`NamedEntity`/`BaseEntity` superclasses.

---

## 7. Change / extension guide

- **Add a vet (runtime)** → **not supported today.** Either seed via `INSERT INTO vets …` in all
  three `data.sql`, or (architectural change) widen `VetRepository` to expose `save`, add a
  controller/form, **and** annotate the write with `@CacheEvict("vets")` (§3.5) — omitting the evict
  makes `/vets` and `/vets.html` serve stale data **silently**.
- **Add a specialty** → `INSERT INTO specialties …` in all three `data.sql`, then link via
  `vet_specialties` rows. No admin surface exists.
- **Link a vet to a specialty** → add `INSERT INTO vet_specialties VALUES (<vet_id>,
  <specialty_id>);` to all three `data.sql`. On H2 there is no unique constraint, so duplicate links
  are accepted **silently** and inflate `getNrOfSpecialties()`.
- **Add a vet field** → `@Column` on `Vet.java` + column in all three `schema.sql`; JSON picks it up
  automatically, XML needs a JAXB annotation. `ddl-auto=none` means a missing column fails **loudly**.
- **Change specialty display order** → edit the `Comparator` in `Vet.getSpecialties()`
  (`Vet.java:59-64`).
- **Change page size / paginate `/vets`** → edit the literal `5` (`VetController.java:60`); to page
  the resource endpoint, switch it to the `Pageable` `findAll`.

---

## 8. Assumptions, Blockers & Open Questions (ABQ)

- **A1 (assumption).** Default runtime profile is H2 (`application.properties:1`); the MySQL/Postgres
  `UNIQUE(vet_id, specialty_id)` and case behaviours are inferred from their `schema.sql` and not
  exercised by default.
- **A2 (assumption).** The `vets` cache provider is whatever JCache implementation is on the
  classpath; `system/CacheConfiguration` only sets `statisticsEnabled` and relies on the provider
  for sizing/eviction defaults (`CacheConfiguration.java:40-51`). The concrete provider lives outside
  this component's path and its eviction policy is **`Unverifiable — Missing Source Evidence`** from
  the `vet` package alone.
- **O1 (open question, out of code scope).** The CAKE catalog for tenant `5K4DVCTX` models an
  appointment-scheduling / provider-availability platform (a *different* system) and includes a
  `SystemComponent` **"Veterinarian Management"** whose only recorded follow-up is *"Detail the
  specific actions (Create, Read, Update, Delete) included in 'Veterinarian Management'."* The
  current code implements **Read only** — no create/update/delete exists. Any CRUD expansion is a
  future evolution decision, not current-state, and is deliberately excluded from the model above.
- **B1 (non-blocker note).** No authentication/authorization; both endpoints are anonymous. The
  read-only design plus the narrow `Repository` contract are the only guardrails against vet
  mutation today.

