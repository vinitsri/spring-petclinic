# ADR: Management / Diagnostics Endpoint Exposure Posture

- **Status:** Accepted (retroactive — documents an already-realized decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-008 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Security · **Impact:** medium · **Dependencies:** ADR-CANDIDATE-007 ([no-application-level-authentication](no-application-level-authentication.md))

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The platform exposes management, diagnostics, and health surfaces alongside the application. Because
the app ships with **no application-level authentication**
([no-application-level-authentication](no-application-level-authentication.md)), the exposure of
these surfaces is itself a trust-boundary decision and warrants its own record.

- ✅ **All** Spring Boot Actuator web endpoints are exposed:
  `management.endpoints.web.exposure.include=*`, with an inline comment stating
  *"Don't do this in production, only for development and testing"* —
  `src/main/resources/application.properties:19-21`.
- ✅ Health probes are surfaced to the orchestrator: the deployment enables additional probe paths
  (`management.endpoint.health.probes.add-additional-paths: true`) and wires liveness/readiness to
  `/livez` and `/readyz` — `k8s/petclinic.yml:39-53`.
- ✅ A diagnostics endpoint `/oups` deliberately throws to demonstrate the error page —
  `src/main/java/org/springframework/samples/petclinic/system/CrashController.java:31-35`.
- ✅ Because there is no security filter chain (ADR-CANDIDATE-007), every exposed endpoint is
  reachable **without credentials**.
- ❓ Whether these surfaces are network-restricted (NetworkPolicy, ingress rules, or a separate
  management port) in any real production environment is **not evidenced in-repo** — the manifests
  expose the app via a `NodePort` Service (`k8s/petclinic.yml:6-12`) with no network policy present.

The default H2 datastore path also makes the H2 console a dev/test convenience; combined with wide
Actuator exposure, the unauthenticated management surface is broad by design for developer
ergonomics.

## Decision

**Expose the full set of management/diagnostics endpoints (Actuator `*`, health probes, the `/oups`
demo) for developer and operator ergonomics, explicitly documenting in-config that wide exposure is
a development/test posture and MUST NOT be used as-is in production.** Production hardening (curating
the exposed endpoint set, restricting the management surface to an internal network/port, and — per
ADR-CANDIDATE-007 — adding authentication in front of it) is deferred to the deploying environment
and is called out as required follow-up rather than encoded in the repository.

## Alternatives Considered

1. **Curate a minimal endpoint set now (e.g. expose only `health`, `info`) and disable the H2
   console.**
   *Rejected for this reference/demo posture* — a minimal set is the correct **production**
   configuration, but it removes the diagnostics breadth (metrics, env, mappings, etc.) that makes
   the sample useful for learning and local operation. The decision here is to keep the demo-open
   posture in-repo while flagging the production-minimal set as required hardening (🎯), not to
   pretend the repo is production-configured.

2. **Move management endpoints to a separate secured management port / internal-only network.**
   *Rejected as an in-repo default* — this is a deployment-topology control (management port +
   NetworkPolicy/ingress) that belongs to the target environment, and there is no evidence the
   demo cluster provides it. Recorded as the recommended production control (🎯) rather than a
   change to the sample manifests.

3. **Leave exposure undocumented (rely on framework defaults implicitly).**
   *Rejected* — the wide exposure is an explicit, security-relevant choice; leaving it implicit
   would hide an unauthenticated management surface. Documenting it (in-config comment + this ADR)
   makes the trade-off visible and auditable.

## Consequences

**Positive**
- ✅ Rich diagnostics and health data are available out-of-the-box for development and operation
  (`application.properties:19-21`; probes in `k8s/petclinic.yml:39-53`).
- ✅ The posture is self-documenting — the config states it is dev/test-only
  (`application.properties:20`), so operators are warned at the point of configuration.

**Negative / trade-offs**
- ✅ Every management endpoint is reachable without authentication (compounding ADR-CANDIDATE-007);
  in an internet-facing deployment this is information disclosure and an unauthenticated management
  surface.
- ❓ Whether production restricts these surfaces at the network layer is unverifiable from the repo
  (no NetworkPolicy; `NodePort` Service, `k8s/petclinic.yml:6-12`) — the control, if any, lives
  outside version control.
- The `/oups` endpoint intentionally raises an exception; harmless as a demo but must not be mistaken
  for a fault when observed in logs/monitoring (`CrashController.java:31-35`).

**Pattern relationship**
- **Instantiates** the catalog pattern *Health Probe & Management-Endpoint Exposure*
  (`docs/architecture-inventory/patterns/observability/health-probe-and-management-endpoints.md`,
  Candidate) and shares the trust-boundary context of
  [no-application-level-authentication](no-application-level-authentication.md). No **Pattern
  Drift** — the pattern is described as-is; the *production-minimal* posture is a hardening delta,
  not a contradiction.
- **Pattern Update Proposal:** capture the "dev-open vs production-minimal exposure" split as
  explicit guidance in the observability pattern file so drift can be checked when the app is
  hardened.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | All Actuator web endpoints exposed; config self-labels as dev/test-only | ✅ | `src/main/resources/application.properties:19-21` |
| E2 | Liveness/readiness probes wired to `/livez`, `/readyz`; extra probe paths enabled | ✅ | `k8s/petclinic.yml:39-53` |
| E3 | `/oups` diagnostics endpoint throws by design | ✅ | `src/main/java/.../system/CrashController.java:31-35` |
| E4 | No security filter chain ⇒ management endpoints are unauthenticated | ✅ | ADR-CANDIDATE-007 ([no-application-level-authentication](no-application-level-authentication.md)); absence of `spring-boot-starter-security` |
| E5 | No in-repo network restriction of the management surface | ❓ | `k8s/petclinic.yml:6-12` (`NodePort`, no NetworkPolicy present) |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The wide exposure is intentionally a development/test posture, not the intended production configuration. | The config comment explicitly says so (`application.properties:20`). | If deployed as-is to production, the unauthenticated management surface becomes a live security exposure. | Confirm the production management configuration with the platform/security owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a formal decision owner to this ADR. | Platform architecture team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Are the management endpoints network-restricted (management port, NetworkPolicy, ingress) in production? | Determines whether the unauthenticated surface is actually reachable by untrusted callers. | Unverifiable in-repo; assume unrestricted until proven otherwise and treat as a hardening item. | Platform / security owner |
| Q2 | Which minimal endpoint set (and H2-console disablement) should the production profile pin? | Needed to turn the "don't do this in production" warning into an enforced configuration. | Expose only `health`/`info`; disable the H2 console; restrict management to an internal surface. | Platform / security owner |
