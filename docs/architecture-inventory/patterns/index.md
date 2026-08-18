# Implementation Pattern Catalog — Petclinic Platform

- **Project Key:** VE
- **Project Name:** Petclinic platform
- **Stage:** Architecture discovery — reusable implementation pattern catalog
- **Branch:** `arch-discovery-57f5f6e8-e25d-475f-8aeb-42d3fc5622b6`
- **Date:** 2026-08-17
- **Scope of this catalog:** Reusable, evidence-grounded implementation patterns observed in the single
  `spring-petclinic` deployable. These are **reusable approaches** for future features, specs, ADRs, and
  implementation — **not** project-specific decisions (there are no ADRs in this repository). A pattern
  may later be supported by an ADR; an ADR may instantiate a pattern.

> **What this catalog is / is not.** It records *how the platform already solves recurring problems* so
> those approaches can be reused deliberately and consistently. It does **not** invent patterns,
> queues, services, databases, protocols, or compliance constraints. Every pattern below is grounded in
> source, configuration, or deployment manifests in the `spring-petclinic` tree. Where evidence is weak
> or absent, items are captured in each file's **Assumptions, Blockers & Open Questions** section rather
> than asserted.

## Grounding and sources

Derived primarily from direct reading of the `spring-petclinic` source tree
(`src/main/java/org/springframework/samples/petclinic/**`, `src/main/resources/**`, `k8s/`, `pom.xml`,
`build.gradle`) and synthesised against the prior discovery baselines:

- `docs/architecture-inventory/repo-inventory.md`
- `docs/architecture-inventory/architecture-views.md`
- `docs/architecture-inventory/baselines/architecture-baseline.md`
- `docs/architecture-inventory/baselines/capability-baseline.md`
- `docs/architecture-inventory/baselines/api-inventory.md`
- `docs/architecture-inventory/baselines/ui-inventory.md`
- `docs/architecture-inventory/baselines/user-journeys.md`

## Status semantics

- **Candidate** — reusable approach observed in code, but **no explicit approval metadata or documented
  human approval exists in the repository** to promote it. Every pattern in this catalog is **Candidate**
  because the repository contains **no ADRs and no approval records** (see the source-of-truth note below).
- **Approved** — reserved for patterns backed by explicit repo approval metadata or a documented human
  approval. **None** qualify today.
- **Deprecated** — a pattern the platform has decided to move away from. **None** recorded.

## Evidence strength legend

- **Strong** — directly evidenced by source code, configuration, and/or deployment manifests read statically.
- **Moderate** — evidenced but with an inference step, or evidenced by config/naming with a partial code path.
- **Weak** — naming/partial signal only; the item's uncertainty is carried into the file's ABQ section.

## Source-of-truth note (CAKE catalog vs observed code)

The CAKE catalog for the working tenant (`5K4DVCTX`) holds a **pattern catalog for a different platform**
— a WebPT EMR appointment-scheduling modernization program. Its `ArchitecturePattern` nodes include the
*decorator pattern (HIPAA audit logging)*, *Webpack Module Federation / MFE embedding*, *transactional
outbox*, *event-sourced CQRS*, *strangler fig*, *lock-then-validate*, *GitOps Helmfile blue-green
deployment*, and *cookie-based session validation*. **None of these are realized in the `spring-petclinic`
source tree.** Per the mandatory Source-of-Truth rule, source code wins: those patterns are recorded as
**Future/Intended State (Not Implemented)** for this repository and are **not** entered as observed
patterns below. They are surfaced in the [ABQ section](#assumptions-blockers--open-questions) as a
governance item (pattern-update proposal candidates) — not as current-state catalog entries.

## Pattern index

| Pattern name | Category | Status | Short description | Related ADRs | Related services/repos | Evidence strength | Last updated |
|--------------|----------|--------|-------------------|--------------|------------------------|-------------------|--------------|
| [Layered Monolith with Package-Scoped Modules](domain/layered-monolith-package-modules.md) | domain | Candidate | Single deployable organised into feature packages (controller → repository → schema), each owning its slice of one shared schema. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Aggregate-Root Persistence via Spring Data JPA](data/aggregate-root-persistence.md) | data | Candidate | Child entities are persisted only through their aggregate root's repository, keeping a consistency boundary in one place. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Server-Rendered MVC with Template Views](ui/server-rendered-mvc-views.md) | ui | Candidate | Controllers return server-rendered template view names; the full HTML page is produced per request with no client-side framework. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Post-Redirect-Get Form Handling with Bean Validation](ui/post-redirect-get-form-validation.md) | ui | Candidate | Form POSTs validate server-side; success redirects (PRG), failure re-renders the same form with field errors. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Read-Through In-Process Cache for Read-Mostly Data](data/read-through-in-process-cache.md) | data | Candidate | Declarative `@Cacheable` read-through cache in front of a read-mostly repository, local to the JVM (no networked cache). | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Profile-Based Environment & Datastore Selection](deployment/profile-based-environment-configuration.md) | deployment | Candidate | One artifact runs against multiple datastores by activating a named configuration profile at deploy time. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Script-Based Relational Schema Provisioning](data/script-based-schema-provisioning.md) | data | Candidate | Schema and seed data applied from checked-in per-engine SQL scripts at startup; runtime DDL auto-generation disabled. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Content-Negotiated Dual Representation](integration/content-negotiated-dual-representation.md) | integration | Candidate | The same domain data is served as an HTML view and as a serialized JSON/XML resource selected by content negotiation. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Request-Scoped Internationalization](ui/request-scoped-localization.md) | ui | Candidate | Per-request locale resolution from a query parameter plus session persistence and externalized message bundles. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Layered Uniqueness Enforcement](data/layered-uniqueness-enforcement.md) | data | Candidate | A business uniqueness rule is enforced both by an application pre-check and by a database constraint, with graceful error translation. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Health Probe & Management-Endpoint Exposure](observability/health-probe-and-management-endpoints.md) | observability | Candidate | Framework management endpoints plus dedicated liveness/readiness probes wired to the orchestrator. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Data-Binding Field Allowlist (Mass-Assignment Protection)](security/data-binding-field-allowlist.md) | security | Candidate | Web data binding disallows client-supplied identity/nested fields to prevent mass-assignment tampering. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |
| [Build-Time Quality & Supply-Chain Gates](testing/build-time-quality-and-supply-chain-gates.md) | testing | Candidate | Style, format, no-HTTP, dependency-enforcer, and SBOM checks run as part of the build/CI, failing the build on violation. | None (no ADRs in repo) | `petclinic` / `spring-petclinic` | Strong | 2026-08-17 |

## Governance notes

- **Pattern drift.** No pattern here is **Approved**, so there is currently no approved-pattern baseline to
  violate. When any pattern is promoted to **Approved** (via an ADR or documented approval), an
  implementation that contradicts it should be flagged as drift in the relevant PR and reconciled against
  the pattern file.
- **Pattern update proposals (for PR-based human review).** Reusable approaches that are **not yet realized
  in this repository** but appear in the CAKE catalog for the (different) scheduling platform — e.g.
  transactional outbox, event-sourced CQRS, strangler-fig migration, decorator-based audit logging,
  Module-Federation micro-frontends — are **not** cataloged as current-state patterns here. If the
  scheduling/EMR program becomes in-scope for this platform (open question across all baselines), they
  should be proposed as new pattern entries through PR review, grounded in the code that would implement
  them. Tracked in the ABQ section below.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Every pattern is **Candidate** because the repository contains no ADRs and no approval metadata to promote any pattern to **Approved**. | Verified absence of `docs/adr/`, `docs/decisions/`, or approval records anywhere in the `spring-petclinic` tree (architecture-baseline §7). | If an out-of-repo approval register exists, some patterns should be **Approved** and drift-checked accordingly. | Confirm with the platform architecture owner whether an external decision/approval register governs these patterns. |
| A2 | The observed approaches are genuinely reusable platform patterns, not one-off code. | Each appears as a repeatable, framework-idiomatic construct usable by future features (e.g. PRG+validation across all form controllers). | Over-cataloging could imply reuse guidance the team never intended. | Review the catalog with the platform architecture owner and prune non-reusable entries. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Ownership attribution on every pattern's Service/Boundary guidance and downstream knowledge-graph ingestion. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Does the CAKE scheduling/EMR pattern catalog (transactional outbox, event-sourced CQRS, strangler fig, decorator audit, Module Federation MFE, lock-then-validate, GitOps Helmfile blue-green) represent planned future scope for this platform? | Determines whether those patterns should be added via PR-based pattern-update proposals or treated as an unrelated catalog. | Not implemented in code; recorded as Future/Intended State (Not Implemented). | Product / architecture owner |
| Q2 | Should any Candidate pattern be promoted to **Approved** via an ADR so drift can be enforced? | Approval turns a described approach into a governed constraint that PRs are checked against. | — | Platform architecture owner |
