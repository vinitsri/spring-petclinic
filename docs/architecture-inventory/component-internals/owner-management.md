# Component Internals — `owner-management`

- **Component:** `owner-management`
- **Repo / path:** `repo` → `src/main/java/org/springframework/samples/petclinic/owner`
- **Stage:** Architecture discovery — component-internals working-model (batch 1 of 2)
- **Grounding:** Derived by reading the component's Java source, JPA mappings, validators,
  formatter, and the DDL/seed SQL it depends on. Every load-bearing claim cites a
  `path:symbol` or a schema/migration artifact. Claims that the code cannot settle are marked
  **`Unverifiable — Missing Source Evidence`**.
- **Maintenance contract:** This model is authoritative and reused by downstream phases. Any
  later change to this component's internals MUST update this file in the same change so the
  model never drifts from the code.

> **Evidence legend:** `code` = a Java symbol/annotation · `schema` = a DDL file · `seed` =
> `data.sql` · `config` = a properties/config file. A behaviour that raises an error is **loud**;
> one that is dropped/ignored without an error is **silent**.

---

## 1. Purpose & boundary

`owner-management` is the **customer-facing clinic-records domain** of Spring PetClinic. It owns
the aggregate rooted at **Owner** and everything reachable through it: an owner's **Pets**, each
pet's **Visits**, and the **PetType** reference vocabulary that classifies a pet. It renders the
server-side Thymeleaf forms and list/detail views for these entities and persists them through
Spring Data JPA.

**It is responsible for:**

- The **Owner aggregate**: create / find (paginated, by last-name prefix) / view / update owners
  (`OwnerController.java`).
- **Pets** as children of an owner: add / edit a pet, including per-owner name-uniqueness and
  birth-date rules (`PetController.java`, `PetValidator.java`).
- **Visits** as children of a pet: book a visit against a pet (`VisitController.java`).
- **PetType** as a **read-only lookup vocabulary** used to type a pet and to bind the type
  dropdown (`PetType.java`, `PetTypeRepository.java`, `PetTypeFormatter.java`).
- The persistence contract for the `owners`, `pets`, `visits`, and `types` tables (schema in
  `src/main/resources/db/{h2,mysql,postgres}/schema.sql`).

**It is explicitly NOT responsible for:**

- **Veterinarian / specialty data** — owned by `vet-management` (`../vet`), which owns the `vets`,
  `specialties`, and `vet_specialties` tables. `owner-management` never reads or writes them.
- **Creating or editing PetType rows** — there is no create/update/delete path for `types` in this
  component; the vocabulary is seed-only (see §3 PetType and §5).
- **Standalone Pet/Visit persistence** — there is **no `PetRepository` and no `VisitRepository`**.
  Pets and Visits are never saved on their own; they are cascaded through the Owner aggregate
  (`Owner.java:64` `@OneToMany(cascade = ALL)`, `Pet.java:56` `@OneToMany(cascade = ALL)`).
- **Authentication / authorization / multi-tenancy** — none exists in this component; every path
  is anonymous and global.
- **Schema migration tooling** — there is no Flyway/Liquibase. Schema is applied by Spring's
  `spring.sql.init` at startup (see §5).

---

## 2. Core concepts (exhaustive)

Each concept below is owned or defined by this component and is expanded in §3.

1. **Owner** — aggregate root; a clinic customer.
2. **Pet** — a pet belonging to exactly one owner; child entity of the Owner aggregate.
3. **PetType** — the reference vocabulary (cat, dog, …) that classifies a Pet; a read-only lookup.
4. **Visit** — a dated appointment/record attached to a Pet; grandchild of the Owner aggregate.
5. **Owner→Pet→Visit aggregate & cascade persistence** — the write model: everything is saved
   through `OwnerRepository`.
6. **Per-owner Pet-name uniqueness** — the invariant `unique_owner_pet_name` and its two-layer
   enforcement.
7. **Telephone format rule** — the 10-digit owner phone constraint.
8. **Person identity fields** — `firstName` / `lastName`, inherited from `model.Person`.
9. **Entity identity & `isNew()` semantics** — how new vs. persistent entities are distinguished.
10. **Last-name prefix search + pagination** — the owner lookup/list mechanism (page size 5).
11. **PetType binding (parse/print)** — how a submitted type string becomes a `PetType`.
12. **Temporal rules** — pet birth-date-not-future and visit-date-must-be-future.
13. **Mass-assignment protection** — the `id` field-binding blacklist.
14. **Post/Redirect/Get + flash messaging** — the request-completion pattern for mutations.

---

## 3. Per concept

### 3.1 Owner

- **Definition** — A clinic customer, the **aggregate root** of this component. Holds contact
  details (`address`, `city`, `telephone`) plus inherited `firstName`/`lastName`, and owns a
  collection of `Pet`s.
- **Representation & storage** — `Owner.java:47-49` `@Entity @Table(name = "owners")`, extending
  `model.Person`. Columns: `address`, `city`, `telephone` (`Owner.java:51-62`); `first_name`,
  `last_name` from `Person.java:31-39`; `id` from `BaseEntity.java:35-37`. Table `owners`
  (`schema.sql`): `id` identity PK, `first_name`, `last_name`, `address`, `city`, `telephone`,
  plus index `owners_last_name`. In H2 `last_name` is `VARCHAR_IGNORECASE(30)` (`db/h2/schema.sql:39`)
  so last-name search is case-insensitive there; MySQL/Postgres use plain `VARCHAR`/`TEXT`.
- **Lifecycle** —
  - *Create:* `OwnerController.processCreationForm` (`OwnerController.java:77-87`) on
    `POST /owners/new`; a fresh `Owner` is supplied by `@ModelAttribute("owner") findOwner` with a
    `null` id (`OwnerController.java:64-70`), validated `@Valid`, then `owners.save(owner)`.
  - *Read:* `findById` (`OwnerRepository.java:60`) for detail/edit; `findByLastNameStartingWith`
    for search (`OwnerRepository.java:45`). Detail view is `showOwner` (`OwnerController.java:169-177`).
  - *Update:* `processUpdateOwnerForm` (`OwnerController.java:144-162`) on `POST /owners/{ownerId}/edit`;
    forces `owner.setId(ownerId)` then `owners.save`.
  - *Delete:* **no delete path exists** for Owner in this component.
  - *Actor:* anonymous HTTP user via Thymeleaf forms. Persistence is `JpaRepository.save`
    (Spring Data), which INSERTs when `isNew()` else UPDATEs.
- **Invariants & enforcement** — Bean Validation on the entity: `address` `@NotBlank`,
  `city` `@NotBlank`, `telephone` `@NotBlank @Pattern(\d{10})` (`Owner.java:52,56,60-62`);
  `firstName`/`lastName` `@NotBlank @Size(max=30)` (`Person.java:32-38`). Violations surface
  **loudly** as `BindingResult` errors and re-render the form (`OwnerController.java:79-82,147-150`).
  The update path additionally rejects a form-vs-URL id mismatch **loudly**
  (`OwnerController.java:152-156`, error code `mismatch`).
- **Extension procedure** — To add an owner field: add the mapped `@Column` field + accessors to
  `Owner.java`, add the column to **all three** `schema.sql` files (`db/h2`, `db/mysql`,
  `db/postgres`), add any Bean Validation annotation for enforcement, and surface it in
  `templates/owners/createOrUpdateOwnerForm.html`. Because `ddl-auto=none` (`application.properties:11`),
  a field with no matching DDL column will **fail loudly** at query time (Hibernate/SQL error), not
  silently.
- **Failure modes** — `findOwner`/`showOwner` throw `IllegalArgumentException("Owner not found
  with id: …")` for an unknown id (`OwnerController.java:67-69,173-174`). A blank required field or
  a non-10-digit phone re-renders the form with field errors and a flash `error` message.

### 3.2 Pet

- **Definition** — A pet belonging to exactly one Owner; a **child entity** with no independent
  repository. Carries `name` (from `NamedEntity`), `birthDate`, a `PetType`, and a set of `Visit`s.
- **Representation & storage** — `Pet.java:44-45` `@Entity @Table(name = "pets")` extending
  `model.NamedEntity` (`name` column, `NamedEntity.java:33-35`). Fields: `birthDate`
  (`@Column @DateTimeFormat(yyyy-MM-dd)`, `Pet.java:48-50`), `type` (`@ManyToOne @JoinColumn(type_id)`,
  `Pet.java:52-54`), `visits` (`@OneToMany(cascade=ALL, EAGER) @JoinColumn(pet_id) @OrderBy("date ASC")`,
  `Pet.java:56-59`). The parent link is **owner-side**: `Owner.pets` is
  `@OneToMany(cascade=ALL, EAGER) @JoinColumn(owner_id) @OrderBy("name")` (`Owner.java:64-67`) — the
  FK `pets.owner_id` is written by the owner mapping, so there is no `owner` field on `Pet`. Table
  `pets`: `id`, `name`, `birth_date`, `type_id NOT NULL` (FK→`types`), `owner_id` (FK→`owners`),
  unique `(owner_id, name)` (`schema.sql`).
- **Lifecycle** —
  - *Create:* `PetController.processCreationForm` (`PetController.java:107-137`) on
    `POST /owners/{ownerId}/pets/new`. `owner.addPet(pet)` (`Owner.java:97-101`, only adds if
    `pet.isNew()`), then `owners.saveAndFlush(owner)` cascades the INSERT.
  - *Read:* via the parent owner — `Owner.getPet(id)` / `getPet(name, ignoreNew)`
    (`Owner.java:117-145`). There is no direct pet query.
  - *Update:* `processUpdateForm` → `updatePetDetails` (`PetController.java:144-200`) mutates the
    **existing** managed pet's fields (`setName/setBirthDate/setType`) and `saveAndFlush`.
  - *Delete:* **no delete path.**
  - *Actor:* anonymous user; persistence always through `OwnerRepository` (cascade).
- **Invariants & enforcement** — Enforced by `PetValidator` (`PetValidator.java:37-54`, wired in
  `PetController.initPetBinder` `PetController.java:94-98`): `name` required; `type` required **only
  when `isNew()`**; `birthDate` required. Plus controller-level rules: name-uniqueness (see §3.6)
  and birth-date-not-future (see §3.12). All surface **loudly** as `BindingResult` field errors.
  Note `type_id` is `NOT NULL` in DDL, so a missing type on an existing pet that slipped validation
  would fail **loudly** at the DB.
- **Extension procedure** — Add a `@Column` field + accessor to `Pet.java`; add the column to all
  three `schema.sql`; add validation in `PetValidator.validate` (this component validates pets in
  **Java, not annotations** — see the class Javadoc `PetValidator.java:24-27`); surface it in
  `templates/pets/createOrUpdatePetForm.html`. A new required field NOT added to `PetValidator`
  will be **silently** accepted as blank.
- **Failure modes** — Adding a pet whose name already exists for the owner is rejected (see §3.6).
  Persisting a pet with no type violates `type_id NOT NULL` **loudly**.

### 3.3 PetType

- **Definition** — The controlled vocabulary that classifies a pet ("cat", "dog", "lizard",
  "snake", "bird", "hamster"). A **read-only lookup** within this component.
- **Representation & storage** — `PetType.java:26-30` `@Entity @Table(name = "types")` extending
  `NamedEntity` (just an `id` + `name`). Table `types`: `id` identity PK, `name VARCHAR(80)`, index
  `types_name` (`schema.sql`). Rows are **seeded**, not created at runtime, by `data.sql`
  (`db/h2/data.sql:18-23` inserts the six types).
- **Lifecycle** —
  - *Create/Update/Delete:* **none at runtime.** The only way a new type enters the system is a new
    `INSERT INTO types …` row in `data.sql` (or a manual DB insert). There is no admin endpoint and
    no `save` call against types anywhere in the component.
  - *Read:* `PetTypeRepository.findPetTypes()` — a `@Query("SELECT ptype FROM PetType ptype ORDER
    BY ptype.name")` (`PetTypeRepository.java:36-37`). Used to populate the dropdown
    (`PetController.populatePetTypes` `PetController.java:62-65`) and to resolve a submitted type
    string (`PetTypeFormatter.parse` `PetTypeFormatter.java:52-60`).
  - *Actor:* seed script / DBA for writes; controllers + formatter for reads.
- **Invariants & enforcement** — `name` is `@NotBlank` (via `NamedEntity`). There is **no uniqueness
  constraint** on `types.name` in any schema, so duplicate type names are possible and would make
  `PetTypeFormatter.parse` bind to the first match by insertion order — an ordering subtlety, not an
  error.
- **Extension procedure** — To add a new pet type: add `INSERT INTO types VALUES (default, '<name>');`
  to **each** `db/{h2,mysql,postgres}/data.sql`. It then appears in the dropdown and is parseable.
  If you instead only add it to a form's option list without a `types` row, `PetTypeFormatter.parse`
  will **throw `ParseException("type not found: …")`** (loud) when that value is submitted
  (`PetTypeFormatter.java:59`).
- **Failure modes** — Submitting a type string that has no `types` row → `ParseException` at bind
  time → field binding error on the form. An empty `types` table → the dropdown is empty and no pet
  can be typed (creation blocked by the `type` required rule).

### 3.4 Visit

- **Definition** — A dated clinic record ("rabies shot", …) attached to a single Pet; a grandchild
  in the Owner aggregate.
- **Representation & storage** — `Visit.java:34-36` `@Entity @Table(name = "visits")` extending
  `model.BaseEntity`. Fields: `date` (`@Column(name="visit_date") @DateTimeFormat(yyyy-MM-dd)`,
  `Visit.java:38-40`) and `description` (`@NotBlank`, `Visit.java:42-43`). The FK `visits.pet_id` is
  written by the **pet-side** mapping (`Pet.java:56-57` `@JoinColumn(name = "pet_id")`); there is no
  `pet` field on `Visit`. Table `visits`: `id`, `pet_id` (FK→`pets`), `visit_date`, `description`,
  index `visits_pet_id` (`schema.sql`). New `Visit()` defaults `date` to **tomorrow**
  (`Visit.java:48-50`, `LocalDate.now().plusDays(1)`).
- **Lifecycle** —
  - *Create:* `VisitController.processNewVisitForm` (`VisitController.java:97-112`) on
    `POST /owners/{ownerId}/pets/{petId}/visits/new`. `loadPetWithVisit`
    (`VisitController.java:63-81`) pre-attaches a new `Visit` to the pet via `pet.addVisit(visit)`;
    the handler then `owner.addVisit(petId, visit)` (`Owner.java:164-174`, asserts pet exists) and
    `owners.save(owner)` cascades the INSERT.
  - *Read:* through `Pet.getVisits()` (ordered by date asc, `Pet.java:58`), rendered on the owner
    detail view.
  - *Update / Delete:* **none.** Visits are append-only through this component.
  - *Actor:* anonymous user; persistence through `OwnerRepository`.
- **Invariants & enforcement** — `description` `@NotBlank` (loud form error). `date` must be strictly
  **after today** (see §3.12), enforced in the controller (`VisitController.java:100-102`), loud.
- **Extension procedure** — Add a `@Column` field to `Visit.java` + column to all three schemas;
  add validation in `processNewVisitForm` or on the entity; surface it in
  `templates/pets/createOrUpdateVisitForm.html`.
- **Failure modes** — Booking against a pet id not belonging to the URL owner throws
  `IllegalArgumentException` in `loadPetWithVisit` (`VisitController.java:71-74`), loud. A blank
  description or a non-future date re-renders the form.

### 3.5 Owner→Pet→Visit aggregate & cascade persistence (the write model)

- **Definition** — The rule that **`Owner` is the only persistence entry point**. Pets and Visits
  are never saved directly; they are reached and written through the owner.
- **Representation & storage** — `Owner.pets` `@OneToMany(cascade = CascadeType.ALL, EAGER)`
  (`Owner.java:64`) and `Pet.visits` `@OneToMany(cascade = CascadeType.ALL, EAGER)` (`Pet.java:56`).
  Both use a **unidirectional `@JoinColumn`** (`owner_id`, `pet_id`) rather than a `mappedBy`
  back-reference. `EAGER` + `spring.jpa.properties.hibernate.default_batch_fetch_size=16`
  (`application.properties:13`) means an owner load pulls its pets and their visits in batched
  queries. `spring.jpa.open-in-view=false` (`application.properties:11`) means lazy access outside a
  transaction would fail — hence the eager collections.
- **Lifecycle** — Every mutation calls `OwnerRepository.save` or `saveAndFlush`
  (`OwnerController.java:84,159`; `PetController.java:126,199`; `VisitController.java:109`). The Pet
  and Visit INSERT/UPDATE/DELETE statements are generated by Hibernate's cascade, not by any pet/visit
  repository (there are none). `saveAndFlush` is used on pet writes specifically so the
  `DataIntegrityViolationException` from the unique constraint surfaces synchronously inside the
  try/catch (`PetController.java:124-134`).
- **Invariants & enforcement** — `Owner.addPet` only adds a pet when `pet.isNew()`
  (`Owner.java:97-101`), preventing duplicate collection entries on re-submits. `updatePetDetails`
  mutates the managed instance in place rather than adding (`PetController.java:186-200`) so an edit
  does not create a second row.
- **Extension procedure** — New child entities of Owner/Pet should follow the same pattern: a
  `@OneToMany(cascade=ALL) @JoinColumn(...)` collection on the parent and **no** dedicated repository,
  or introduce a repository if the child needs independent querying. Downstream code that adds a new
  child and expects it to persist **must** route the save through `OwnerRepository`; a bare `new`
  child never persisted through the owner is **silently** dropped.
- **Failure modes** — Forgetting to `save`/`saveAndFlush` the owner after mutating a child leaves
  the change in memory only (silent no-op). Accessing a collection after the transaction with
  `open-in-view=false` would throw `LazyInitializationException` — avoided here by `EAGER`.

### 3.6 Per-owner Pet-name uniqueness

- **Definition** — Within one owner, two pets may not share a (case-insensitive) name.
- **Representation & storage** — DB constraint `unique_owner_pet_name` on `(owner_id, name)`:
  `ALTER TABLE pets ADD CONSTRAINT unique_owner_pet_name UNIQUE (owner_id, name)` in H2
  (`db/h2/schema.sql:55`) and MySQL (`db/mysql/schema.sql:47`); in Postgres it is a **functional
  unique index on `(owner_id, LOWER(name))`** (`db/postgres/schema.sql:45`). Case-insensitivity in
  H2/MySQL comes from `VARCHAR_IGNORECASE`/collation; in Postgres from the explicit `LOWER()`.
- **Lifecycle / enforcement (two layers, both loud)** —
  1. **Application pre-check.** On create, `owner.getPet(pet.getName(), true) != null` →
     `result.rejectValue("name","duplicate","already exists")` (`PetController.java:111-113`). On
     edit, `owner.getPet(petName, false)` where the found pet's id differs → same rejection
     (`PetController.java:151-156`). `getPet(name, ignoreNew)` matches case-insensitively
     (`Owner.java:135-145`, `equalsIgnoreCase`).
  2. **Database backstop.** If a concurrent/edge case slips past the pre-check, `saveAndFlush`
     raises `DataIntegrityViolationException`; the controller inspects the message for
     `"unique_owner_pet_name"` via `isDuplicatePetNameViolation` (`PetController.java:202-205`) and,
     if matched, converts it to the same `"name" / "duplicate"` field error; **any other**
     integrity violation is re-thrown (`PetController.java:128-131,170-173`).
- **Extension procedure** — To change the uniqueness scope (e.g. make names globally unique or add
  a soft-delete dimension) you must change the constraint in **all three** schema files AND the
  `getPet(...)` pre-check AND the `isDuplicatePetNameViolation` message match. Changing only the DB
  constraint leaves the app pre-check enforcing the old rule; changing only the app check means the
  DB backstop (matched by the literal string `unique_owner_pet_name`) still governs.
- **Failure modes** — If the constraint were renamed without updating `isDuplicatePetNameViolation`,
  a duplicate would surface as an **uncaught** `DataIntegrityViolationException` (500) instead of a
  friendly field error — a silent-to-loud regression.

### 3.7 Telephone format rule

- **Definition** — An owner's telephone must be exactly 10 digits.
- **Representation & storage** — `@Pattern(regexp = "\\d{10}", message = "{telephone.invalid}")`
  on `Owner.telephone` (`Owner.java:60-62`), plus `@NotBlank`. Stored as `telephone VARCHAR(20)`
  (H2/MySQL) / `TEXT` (Postgres). The message key `telephone.invalid` resolves via
  `spring.messages.basename=messages/messages` (`application.properties`).
- **Lifecycle / enforcement** — Evaluated by `@Valid` on owner create/update
  (`OwnerController.java:78,145`); a non-match is a **loud** `BindingResult` error re-rendering the
  form. The DB column width (20) is looser than the rule (10) — the app rule is the real constraint.
- **Extension procedure** — Change the regex/message in `Owner.java`; update `telephone.invalid` in
  the `messages_*.properties` bundles. No schema change needed unless length grows past 20.
- **Failure modes** — Any non-10-digit value (letters, spaces, +country code) is rejected loudly.

### 3.8 Person identity fields (`firstName` / `lastName`)

- **Definition** — Human name fields shared by `Owner` (and by `Vet` in the other component) via the
  `model.Person` mapped superclass.
- **Representation & storage** — `Person.java:31-39`: both `@Column(length = 30) @Size(max = 30)
  @NotBlank`. Columns `first_name`/`last_name VARCHAR(30)`. `owners.last_name` is indexed
  (`owners_last_name`) and, in H2, `VARCHAR_IGNORECASE`.
- **Lifecycle / enforcement** — Bound from the owner form; `@NotBlank`/`@Size` enforced by `@Valid`
  (loud). `last_name` drives the search in §3.10.
- **Extension procedure** — This is a **shared** superclass in `model/`; changing it affects both
  `owner-management` and `vet-management`. Prefer local fields on `Owner` for owner-only additions.
- **Failure modes** — Blank or >30-char names rejected loudly.

### 3.9 Entity identity & `isNew()` semantics

- **Definition** — How the component tells a not-yet-persisted entity from a persistent one.
- **Representation & storage** — `BaseEntity` (`model/BaseEntity.java:33-49`): `@Id @GeneratedValue
  (strategy = IDENTITY)` integer `id`; `isNew()` returns `id == null`. All four entities
  (Owner/Pet/PetType/Visit) inherit this. `GenerationType.IDENTITY` maps to the DB identity/
  auto-increment columns (`GENERATED BY DEFAULT AS IDENTITY` / `AUTO_INCREMENT`).
- **Lifecycle / enforcement** — `isNew()` gates behaviour: `Owner.addPet` only appends new pets
  (`Owner.java:98`); `PetValidator` requires `type` only for new pets (`PetValidator.java:46`);
  duplicate-name pre-checks use `ignoreNew` accordingly (`Owner.java:139`). `save` INSERTs when new,
  UPDATEs otherwise (Spring Data default via `isNew`).
- **Extension procedure** — Do not set `id` from user input — it is blacklisted from binding (§3.13).
  New entities should rely on DB-generated identity; do not assign ids manually.
- **Failure modes** — Because identity is DB-generated, `owner.getId()` is null until after
  `save`/flush; the create flow redirects using `owner.getId()` **after** save
  (`OwnerController.java:86`).

### 3.10 Last-name prefix search + pagination

- **Definition** — The owner lookup: find owners whose last name **starts with** a query, paged 5
  per page.
- **Representation & storage** — Derived query `findByLastNameStartingWith(String, Pageable)`
  (`OwnerRepository.java:45`) — Spring Data translates the method name to a `LIKE 'q%'`. Page size
  is hard-coded `5` in `findPaginatedForOwnersLastName` (`OwnerController.java:133-137`).
- **Lifecycle / control flow** — `GET /owners` → `processFindForm` (`OwnerController.java:94-122`):
  a `null` last name is coerced to `""` (matches everything), otherwise `.strip()`ed;
  `findByLastNameStartingWith` runs; **0 results** → `rejectValue("lastName","notFound","not
  found")` and re-render the find form; **exactly 1** → redirect to that owner's detail; **>1** →
  render the paginated list (`ownersList`) via `addPaginationModel` (`OwnerController.java:124-131`).
- **Invariants & enforcement** — Case-insensitivity depends on the DB: H2 `VARCHAR_IGNORECASE`
  makes it case-insensitive; on MySQL it depends on column collation; on Postgres (`TEXT`, no
  `LOWER`) the prefix match is **case-sensitive**. This is an environment-dependent behaviour, not
  an app-enforced invariant.
- **Extension procedure** — To search other fields, add a derived/`@Query` method to
  `OwnerRepository` and call it from the controller. To change page size, edit the literal `5`
  (`OwnerController.java:134`).
- **Failure modes** — A `page` param below 1 yields `PageRequest.of(page-1, …)` with a negative
  index → `IllegalArgumentException` (loud). No results is handled gracefully (form error).

### 3.11 PetType binding (parse / print)

- **Definition** — How the string submitted in the type dropdown is converted to a `PetType`
  entity, and how a `PetType` renders back to text.
- **Representation & storage** — `PetTypeFormatter implements Formatter<PetType>`
  (`PetTypeFormatter.java:37`), a Spring `@Component` registered globally (via
  `system/WebConfiguration`). `print` returns `petType.getName()` (`:46-49`). `parse` loads
  `types.findPetTypes()` and returns the first whose `name` equals the text, else throws
  `ParseException` (`:52-60`).
- **Lifecycle / enforcement** — Runs during form binding whenever a `PetType` field is bound. An
  unknown type is a **loud** bind error. The dropdown options themselves come from
  `PetController.populatePetTypes` (`PetController.java:62-65`).
- **Extension procedure** — See §3.3: new types require a `types` seed row so `parse` can resolve
  them. The formatter is keyed on **exact name equality** (`Objects.equals`), so option labels must
  match the stored `name` verbatim.
- **Failure modes** — Case/whitespace mismatch between the submitted value and the stored `name`
  → `ParseException` → bind error, even though a "similar" type exists.

### 3.12 Temporal rules (birth date, visit date)

- **Definition** — A pet's birth date may not be in the future; a visit's date must be strictly in
  the future.
- **Representation & storage** — Pure controller logic against `LocalDate.now()`; no schema column
  constraint. Pet: `PetController.java:115-118` (create) and `:158-161` (edit) reject
  `birthDate.isAfter(today)` with code `typeMismatch.birthDate`. Visit: `VisitController.java:100-102`
  rejects `!date.isAfter(today)` with code `typeMismatch.visitDate`; the visit form also exposes a
  `minVisitDate` model attribute = tomorrow (`VisitController.java:83-86`).
- **Lifecycle / enforcement** — Evaluated on every pet/visit submit; **loud** field errors. Note
  the required-ness of `birthDate` is separately enforced by `PetValidator` (§3.2).
- **Extension procedure** — Adjust the comparison in the respective controller; add/adjust the
  message key (`typeMismatch.birthDate` / `typeMismatch.visitDate`) in the message bundles.
- **Failure modes** — A future birth date or a today/past visit date re-renders the form.

### 3.13 Mass-assignment protection (`id` blacklist)

- **Definition** — Preventing a client from setting an entity `id` (or nested `*.id`) through form
  binding.
- **Representation & storage** — `WebDataBinder.setDisallowedFields("id", "*.id")` in every
  controller's `@InitBinder`: `OwnerController.setAllowedFields` (`OwnerController.java:59-62`),
  `PetController.initOwnerBinder`/`initPetBinder` (`PetController.java:89-98`),
  `VisitController.setAllowedFields` (`VisitController.java:51-54`).
- **Lifecycle / enforcement** — Any inbound `id`/`*.id` request parameter is **silently ignored**
  during binding (not an error). Ids therefore come only from the URL path (`{ownerId}`, `{petId}`)
  or DB generation. The owner update path re-asserts identity from the path and additionally checks
  form-vs-URL id equality (`OwnerController.java:152-158`).
- **Extension procedure** — Any new controller in this component that binds an entity must repeat
  the `setDisallowedFields("id","*.id")` init-binder, or it reopens the mass-assignment hole.
- **Failure modes** — Omitting the blacklist would let a client overwrite an arbitrary id — silent
  data corruption. Present code prevents it.

### 3.14 Post/Redirect/Get + flash messaging

- **Definition** — The pattern that every successful mutation ends in a redirect carrying a
  one-shot flash message, so a browser refresh does not re-submit.
- **Representation & storage** — Handlers take `RedirectAttributes` and, on success,
  `addFlashAttribute("message", …)` then return `redirect:/owners/{ownerId}` (or `/owners/{id}`):
  `OwnerController.java:85-86,160-161`; `PetController.java:135-136,177-178`;
  `VisitController.java:110-111`.
- **Lifecycle / enforcement** — Flash attributes survive exactly one redirect. **Note a quirk:** on
  the owner *error* branches the code adds a flash `error` but returns a **view name** rather than a
  redirect (`OwnerController.java:79-82,147-150`); flash attributes are only reliably consumed
  across a redirect, so that particular error flash is effectively a no-op — the field errors on the
  re-rendered form are what the user sees. This is observed behaviour, not a defect this stage fixes.
- **Extension procedure** — New mutating handlers should follow the same PRG shape: validate → on
  error re-render the form view; on success `save`, add a flash `message`, and `redirect:`.
- **Failure modes** — Returning a view (not a redirect) after a write would re-submit on refresh;
  the success paths all redirect, avoiding this.

---

## 4. Primary control flows

All flows are server-rendered Spring MVC (Thymeleaf); there is no session state
(`open-in-view=false`, no `@SessionAttributes`). Entry points are the `@GetMapping`/`@PostMapping`
handlers.

**A. Create owner** — `GET /owners/new` → `initCreationForm` returns the form
(`OwnerController.java:72-75`). `POST /owners/new` → `processCreationForm`: `@Valid Owner` (fresh,
from `findOwner` with null id) → on error re-render form; else `owners.save(owner)` →
`redirect:/owners/{newId}` with flash `message` (`OwnerController.java:77-87`).

**B. Find / list owners** — `GET /owners/find` → `initFindForm` (`OwnerController.java:89-92`).
`GET /owners` → `processFindForm`: normalise last name → `findByLastNameStartingWith(page-1, size
5)` → 0 ⇒ form error; 1 ⇒ redirect to detail; N ⇒ `ownersList` view with pagination model
(`OwnerController.java:94-131`, §3.10).

**C. Show owner** — `GET /owners/{ownerId}` → `showOwner`: `owners.findById` (or throw) →
`ModelAndView("owners/ownerDetails")` with the owner (and, eagerly, its pets and their visits)
(`OwnerController.java:169-177`).

**D. Update owner** — `GET /owners/{ownerId}/edit` → form (`OwnerController.java:139-142`). `POST`
→ `processUpdateOwnerForm`: `@Valid` → on error re-render; else id-mismatch guard → `setId(ownerId)`
→ `owners.save` → `redirect:/owners/{ownerId}` (`OwnerController.java:144-162`).

**E. Add pet** — `GET /owners/{ownerId}/pets/new` → `initCreationForm` seeds a new pet on the owner
(`PetController.java:100-105`). `POST` → `processCreationForm`: duplicate-name pre-check → future
birth-date check → `PetValidator` (via `@Valid`) → `owner.addPet` → `owners.saveAndFlush` inside a
try/catch that maps `unique_owner_pet_name` violations to a field error → redirect
(`PetController.java:107-137`, §3.6).

**F. Edit pet** — `GET /owners/{ownerId}/pets/{petId}/edit` → form (`PetController.java:139-142`).
`POST` → `processUpdateForm`: duplicate-name (excluding self) → future birth-date → `@Valid` →
`updatePetDetails` mutates the managed pet and `saveAndFlush` (same violation handling) → redirect
(`PetController.java:144-200`).

**G. Book visit** — `GET /owners/{ownerId}/pets/{petId}/visits/new` → `loadPetWithVisit` attaches a
new (tomorrow-dated) visit and returns the form (`VisitController.java:63-93`). `POST` →
`processNewVisitForm`: future-date check → `@Valid` (description not blank) → `owner.addVisit(petId,
visit)` → `owners.save` → redirect (`VisitController.java:97-112`).

**H. Type-dropdown binding (cross-cutting)** — On any pet form, `populatePetTypes` supplies options
(`PetController.java:62-65`) and `PetTypeFormatter` converts the selected string to a `PetType`
during binding (§3.11).

---

## 5. Persistence & schema evolution

- **Datastore.** Relational, via Spring Data JPA / Hibernate. Default profile is **H2 in-memory**
  (`application.properties:1` `database=h2`); MySQL and Postgres are supported by switching the
  `database` property, which selects `classpath*:db/${database}/schema.sql` and `.../data.sql`
  (`application.properties:2-4`).
- **Tables owned by this component:** `owners`, `pets`, `types`, `visits` (plus the FKs
  `pets.owner_id`, `pets.type_id`, `visits.pet_id`). Defined in
  `src/main/resources/db/{h2,mysql,postgres}/schema.sql`.
- **How schema changes are applied.** There is **no migration framework** (no Flyway/Liquibase on
  the classpath; `build.gradle`/`pom.xml` show none). `spring.jpa.hibernate.ddl-auto=none`
  (`application.properties:11`) — Hibernate does **not** create or alter tables. Instead Spring's
  `spring.sql.init` runs the selected `schema.sql` then `data.sql` at startup. So a schema change is
  a **hand-edited DDL change replicated across all three dialect files**; there are no versioned
  migrations and no migration count.
- **Seeding & idempotency.** `data.sql` uses plain `INSERT … VALUES (default, …)`
  (`db/h2/data.sql`) with **no `ON CONFLICT`/`MERGE`** — it is **not idempotent**. For H2 the schema
  is dropped-and-recreated each boot (`DROP TABLE … IF EXISTS` at the top of `db/h2/schema.sql:1-7`),
  so re-seeding is clean; MySQL/Postgres use `CREATE TABLE IF NOT EXISTS` and would accumulate
  duplicate seed rows if pointed at a persistent DB across restarts. This is a real operational
  caveat for non-H2 profiles.
- **Dialect divergences that matter (already noted per concept):** case-insensitive columns
  (`VARCHAR_IGNORECASE` in H2 only), and the pet-name uniqueness expressed as `UNIQUE(owner_id,
  name)` in H2/MySQL vs a functional `UNIQUE INDEX (owner_id, LOWER(name))` in Postgres
  (`db/postgres/schema.sql:45`).

---

## 6. Surface → internals map

| Surface (HTTP) | Handler | Internal mechanism driven | Kind |
|---|---|---|---|
| `GET /owners/new` | `OwnerController.initCreationForm` | render empty owner form | read-only |
| `POST /owners/new` | `OwnerController.processCreationForm` | `@Valid` → `OwnerRepository.save` → PRG | **mutating** (insert `owners`) |
| `GET /owners/find` | `OwnerController.initFindForm` | render find form | read-only |
| `GET /owners` | `OwnerController.processFindForm` | `findByLastNameStartingWith` + paging (§3.10) | read-only |
| `GET /owners/{ownerId}` | `OwnerController.showOwner` | `findById` + eager pets/visits | read-only |
| `GET /owners/{ownerId}/edit` | `OwnerController.initUpdateOwnerForm` | render populated form | read-only |
| `POST /owners/{ownerId}/edit` | `OwnerController.processUpdateOwnerForm` | id guard → `save` → PRG | **mutating** (update `owners`) |
| `GET /owners/{ownerId}/pets/new` | `PetController.initCreationForm` | seed new `Pet` on owner | read-only |
| `POST /owners/{ownerId}/pets/new` | `PetController.processCreationForm` | dup-name + birthdate checks → `PetValidator` → `saveAndFlush` (cascade) | **mutating** (insert `pets`) |
| `GET /owners/{ownerId}/pets/{petId}/edit` | `PetController.initUpdateForm` | render pet form | read-only |
| `POST /owners/{ownerId}/pets/{petId}/edit` | `PetController.processUpdateForm` | dup-name(self) + checks → `updatePetDetails` → `saveAndFlush` | **mutating** (update `pets`) |
| `GET /owners/{ownerId}/pets/{petId}/visits/new` | `VisitController.initNewVisitForm` | attach tomorrow-dated `Visit` | read-only |
| `POST /owners/{ownerId}/pets/{petId}/visits/new` | `VisitController.processNewVisitForm` | future-date check → `@Valid` → `owner.addVisit` → `save` (cascade) | **mutating** (insert `visits`) |
| *(no endpoint)* | — | **PetType create/update/delete** | **absent** — seed-only (§3.3) |
| *(no endpoint)* | — | **Owner/Pet/Visit delete** | **absent** — no delete path |

Internal (non-HTTP) surfaces: `PetTypeFormatter` (bean, global type binding, §3.11);
`OwnerRepository` / `PetTypeRepository` (Spring Data, §3.1/§3.3); the `model.Person`/`NamedEntity`/
`BaseEntity` superclasses (shared identity/name/person mapping).

---

## 7. Change / extension guide

- **Add an owner field** → field+accessors on `Owner.java`; column in all three `schema.sql`;
  optional Bean Validation annotation; form field in `createOrUpdateOwnerForm.html`. *Silent
  rejecter:* none for binding, but a missing DDL column fails **loudly** at query time
  (`ddl-auto=none`).
- **Add a pet field** → field on `Pet.java`; column in all three `schema.sql`; rule in
  `PetValidator.validate` (validation here is **Java, not annotations**); form field. *Silent
  rejecter:* a required field not added to `PetValidator` is accepted blank.
- **Add a pet type** → `INSERT INTO types …` in all three `data.sql`. *Silent/loud rejecter:*
  submitting a type with no `types` row throws `ParseException` in `PetTypeFormatter.parse`
  (`:59`) — **loud** bind error. Adding it only to a form option list is insufficient.
- **Change pet-name uniqueness** → constraint in all three `schema.sql` **and** the `getPet(...)`
  pre-check **and** the literal `"unique_owner_pet_name"` match in `isDuplicatePetNameViolation`
  (`PetController.java:204`). Miss any one and enforcement splits between layers.
- **Add a new mutating endpoint** → repeat `setDisallowedFields("id","*.id")` in its `@InitBinder`
  (§3.13) and follow PRG (§3.14); route all persistence through `OwnerRepository` (§3.5).
- **Switch database** → set `database=mysql|postgres` (`application.properties:1`) and beware the
  non-idempotent `data.sql` on persistent stores (§5) and the case-sensitivity divergence in
  last-name search (§3.10) and pet-name uniqueness (§3.6).

---

## 8. Assumptions, Blockers & Open Questions (ABQ)

- **A1 (assumption).** The active runtime profile is H2 (`application.properties:1`); MySQL/Postgres
  behaviour (case sensitivity, seed idempotency) is inferred from their `schema.sql`/`data.sql` and
  is **not** exercised by default config.
- **A2 (assumption).** `PetTypeFormatter` is registered globally via `system/WebConfiguration`
  (the `@Component Formatter` convention); the registration class lives outside this component's
  path and was not re-read here. The formatter's *behaviour* is verified from its own source.
- **O1 (open question).** The owner *error*-branch flash attributes are added but the handler
  returns a view rather than a redirect (`OwnerController.java:79-82,147-150`), so the flash `error`
  is effectively unused. Recorded as observed behaviour; whether this is intended is undetermined
  from code.
- **O2 (open question, out of code scope).** The CAKE catalog for tenant `5K4DVCTX` contains a
  `DecisionRecord` **"VE-39: Owner Pet Preferences Extension"** linked (`AFFECTS_COMPONENT`) to an
  `Owner` component, implying a future "pet preferences" extension. **No such fields, columns, or
  code exist** in the current `owner` package — it is a proposed/backlog decision, not current-state,
  and is deliberately excluded from the model above. Downstream evolution phases own that decision.
- **B1 (non-blocker note).** No authentication/authorization exists in this component; every path is
  anonymous. If a downstream phase adds ownership/tenancy, it must thread it through these handlers
  and the aggregate.


