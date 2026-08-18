# ADR: Kubernetes Deployment via Plain Manifests + Service Binding (No Helm)

- **Status:** Accepted (retroactive — documents an existing, evidenced decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture / deployment team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-011 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Orchestration · **Impact:** medium · **Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-005

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The platform is deployed to Kubernetes using **hand-written plain manifests** applied directly with
`kubectl apply` — there is no Helm chart, no Kustomize overlay, and no GitOps controller.

- ✅ The application is a single `Deployment` (`replicas: 1`) fronted by a `NodePort` `Service`,
  both declared statically in `k8s/petclinic.yml:1-64`.
- ✅ The datastore is a co-deployed PostgreSQL `Deployment` + `Service` plus a
  `servicebinding.io/postgresql` `Secret`, declared in `k8s/db.yml:1-72`.
- ✅ Database credentials reach the app through the **Service Binding** spec, not app config: the
  container sets `SERVICE_BINDING_ROOT=/bindings` and projects the `demo-db` secret at
  `/bindings/secret` (`k8s/petclinic.yml:37-38,55-64`), and the secret carries the
  `servicebinding.io/postgresql` type with connection fields (`k8s/db.yml:3-14`).
- ✅ The workload is pinned to the `postgres` Spring profile (`SPRING_PROFILES_ACTIVE=postgres`,
  `k8s/petclinic.yml:35-36`) — this is why this ADR depends on the multi-engine profile decision
  (ADR-CANDIDATE-005, see [multi-engine-datastore-spring-profiles](multi-engine-datastore-spring-profiles.md)).
- ✅ Liveness/readiness are wired to dedicated probe paths `/livez` and `/readyz`
  (`k8s/petclinic.yml:47-54`), enabled by `management.endpoint.health.probes.add-additional-paths`
  in `SPRING_APPLICATION_JSON` (`k8s/petclinic.yml:39-43`).
- ✅ CI applies the whole directory verbatim — `kubectl apply -f k8s/` — against an ephemeral Kind
  cluster and waits on pod readiness (`.github/workflows/deploy-and-test-cluster.yml:23-30`). No
  templating step precedes the apply.

There is a **single, unparameterised environment**: values such as the image tag (`dsyer/petclinic`,
`k8s/petclinic.yml:33`), replica count, and credentials are literals in the manifests, not templated
inputs. This keeps deployment mechanically simple but offers no per-environment variation.

## Decision

**Deploy the `petclinic` monolith to Kubernetes as plain, statically-declared manifests
(`k8s/*.yml`) applied with `kubectl apply -f k8s/`, injecting datastore credentials through the
`servicebinding.io` Secret projection rather than a templating or packaging tool.** No Helm chart,
Kustomize overlay, or GitOps controller is introduced; the manifests are the single source of the
deployed topology. Because the monolith is one deployable (ADR-CANDIDATE-001), a single
`Deployment`/`Service` pair suffices, and engine selection is delegated to the `postgres` profile
(ADR-CANDIDATE-005).

## Alternatives Considered

1. **Helm chart.**
   *Rejected for the current scope* — Helm adds templating, values-per-environment, and
   release/rollback tracking that would be genuinely useful once more than one environment exists.
   It was not adopted because the platform deploys a **single** unparameterised environment (one
   image, one replica), so a chart's templating machinery would add packaging and lifecycle overhead
   with no variation to template today. Reconsider when a second environment (staging/prod split) or
   real release management is required (see Q1).

2. **Kustomize overlays.**
   *Rejected for now* — Kustomize would give per-environment base+overlay patching without Helm's
   templating language, a lighter-weight fit than Helm. It was not chosen because there is currently
   only one environment to render, so there is nothing to overlay; the raw manifests already ARE the
   base. It becomes the low-friction next step the moment environment-specific values appear.

3. **GitOps (Argo CD / Flux) reconciling the manifests.**
   *Rejected as out of scope here* — continuous reconciliation and drift detection are valuable
   operationally but require cluster-side controllers and a promotion pipeline the repository does
   not define. `kubectl apply` in CI is the imperative equivalent used today; GitOps is a later
   operability investment, not a documentation-stage decision.

## Consequences

**Positive**
- ✅ The deployed topology is fully legible in two short files (`k8s/petclinic.yml`, `k8s/db.yml`) —
  no template indirection to trace.
- ✅ Credential handling is decoupled from application config via the Service Binding projection,
  so the app reads no DB URL/user/password from properties in the cluster
  (`k8s/petclinic.yml:37-38,55-64`; `k8s/db.yml:3-14`).
- ✅ Deployment is CI-verified end-to-end on Kind, including pod readiness gating
  (`.github/workflows/deploy-and-test-cluster.yml:20-30`).

**Negative / trade-offs**
- ✅ No templating means **no per-environment variation**: image tag, replica count, and credentials
  are literals (`k8s/petclinic.yml:22,33`; `k8s/db.yml:7-14`). A second environment requires copying
  and hand-editing manifests — the exact drift risk Helm/Kustomize exist to prevent.
- ✅ Plaintext credentials are committed in the Secret's `stringData` (`k8s/db.yml:7-14`) — acceptable
  for a demo, unacceptable for a real cluster; production requires an external secret source. This
  reinforces the demo trust-boundary posture in
  [no-application-level-authentication](no-application-level-authentication.md).
- ❓ No rollback/release history: `kubectl apply` mutates live objects with no chart revision to roll
  back to. Whether any out-of-repo release tooling wraps this is not evidenced (see Q1).
- ✅ The `NodePort` Service (`k8s/petclinic.yml:7`) exposes the app without an Ingress/LoadBalancer
  abstraction — suitable for Kind/dev, not a managed ingress posture.

**Pattern relationship**
- **Instantiates** the *Profile-Based Environment & Datastore Selection* pattern
  (`patterns/deployment/profile-based-environment-configuration.md`) — the manifest activates the
  `postgres` profile at deploy time rather than baking engine choice into the image.
- **Relates to** the *Health Probe & Management-Endpoint Exposure* pattern
  (`patterns/observability/health-probe-and-management-endpoints.md`) — the `/livez` and `/readyz`
  probes are the orchestrator-facing side of that pattern.
- **No Pattern Drift:** no catalog pattern prescribes a packaging tool (Helm/Kustomize), so plain
  manifests do not contradict an Approved pattern.
- **Pattern Update Proposal:** if a second environment is introduced, add a *Deployment packaging*
  pattern entry recording the chosen overlay/templating approach so it is applied consistently.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Single Deployment + NodePort Service, statically declared | ✅ | `k8s/petclinic.yml:1-64` |
| E2 | Co-deployed PostgreSQL + Service + service-binding Secret | ✅ | `k8s/db.yml:1-72` |
| E3 | Credentials injected via `servicebinding.io` projection, not app config | ✅ | `k8s/petclinic.yml:37-38,55-64`; `k8s/db.yml:3-14` |
| E4 | Deploy applies the directory as-is via `kubectl apply -f k8s/` | ✅ | `.github/workflows/deploy-and-test-cluster.yml:23-30` |
| E5 | Liveness/readiness bound to `/livez` and `/readyz` | ✅ | `k8s/petclinic.yml:39-54` |
| E6 | No Helm chart / Kustomize / GitOps config present | ✅ | absence of `Chart.yaml`, `kustomization.yaml`, or Argo/Flux manifests in the tree |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `k8s/*.yml` is the intended production deployment mechanism, not only a demo/Kind convenience. | The manifests are the only deployment artifacts in-repo and CI exercises them (`deploy-and-test-cluster.yml`). | If production uses a different (out-of-repo) mechanism, this ADR documents only the demo path. | Confirm the production deployment pipeline with the platform deployment owner. |
| A2 | The single unparameterised environment is deliberate, not an unfinished multi-env setup. | Only literal values exist in the manifests; no overlay or values scaffolding is present. | If multi-env was intended, plain manifests are a gap rather than a decision. | Confirm environment strategy with the deployment owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a decision owner to this ADR and to Q1. | Platform architecture / deployment team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | When a second environment (or real release management) is needed, adopt Kustomize overlays or a Helm chart? | Determines the migration path off single-file manifests and how rollback/promotion are handled. | Prefer Kustomize first (lighter than Helm) unless chart distribution is required. | Platform deployment owner |
| Q2 | Where should production DB credentials come from instead of the committed `stringData` Secret? | Committed plaintext credentials are unacceptable outside a demo cluster. | External secret store (e.g. sealed-secrets / cloud secret manager) surfaced through the same service-binding projection. | Platform security owner |
