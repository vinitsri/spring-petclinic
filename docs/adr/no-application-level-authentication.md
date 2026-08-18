# ADR: No Application-Level Authentication or Authorization

- **Status:** Accepted (retroactive — documents an already-realized posture)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture / security owner (attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-007 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Security · **Impact:** high · **Dependencies:** none

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

The platform ships with **no application-level authentication or authorization**. Every endpoint —
domain pages, the `GET /vets` JSON resource, Actuator, and the H2 console — is reachable without
credentials.

- ✅ There is **no** `spring-boot-starter-security` (or equivalent) dependency in `pom.xml` or
  `build.gradle` (`grep security` returns nothing across both build files).
- ✅ There is no security filter chain, `SecurityFilterChain` bean, login flow, session-based auth, or
  role/permission check in any controller or in the `system` package (architecture-baseline §5).
- ✅ Actuator is fully exposed (`management.endpoints.web.exposure.include=*`,
  `application.properties:21`), with an inline comment flagging it as development/testing-only
  ("Don't do this in production", `application.properties:19–21`).

The application is a well-known **reference/sample** Spring application, which supplies a documented
rationale for the absence: it optimizes for demonstration and learning, not for an internet-facing
trust boundary. This is a deliberate posture decision, not a mere oversight — it determines whether
the app can be exposed to untrusted networks and defines exactly what a production-hardening effort
must add.

The `multi-tenant isolation`, `HIPAA audit`, session-based auth, and RLS patterns in the CAKE catalog
(`ADR-MULTI-TENANT-DATA-ISOLATION`, `ADR-SESSION-BASED-AUTHENTICATION`, `ADR-006`, `ADR-PLATFORM-027`)
belong to the WebPT EMR scheduling program, a different platform; none is realized here (Source-of-Truth:
code wins).

## Decision

**Ship with no application-level authentication or authorization** — no Spring Security, no login, no
session auth, no role/permission enforcement — treating the deployable as an unauthenticated
reference application. Any access control, if required for a given deployment, is delegated to an
**external layer** (network policy / ingress / gateway) rather than added in-application. 🎯 Adding
in-app authn/authz (e.g. Spring Security form/OAuth2-OIDC) is the explicit hardening step required
before this app is exposed to untrusted users.

## Alternatives Considered

1. **Spring Security with form or HTTP Basic authentication.**
   *Rejected (for the current state)* — introduces a login flow, credential storage, and session
   management that a demonstration app does not need and that would obscure the domain code the sample
   is meant to teach. The trade-off is accepted knowingly: the app must not be treated as safe to
   expose without an external control.

2. **OAuth2 / OIDC via an external identity provider.**
   *Rejected* — requires standing up or integrating an IdP and configuring token validation, which is
   disproportionate for a self-contained sample with no user/identity model. It is the natural target
   🎯 if the platform ever serves real clinic users.

3. **External gateway/ingress enforcing authentication in front of the app.**
   *Not rejected as a future control, but not present in-repo* — no such gateway is evidenced in this
   clone, so the app is unauthenticated as deployed here. Whether one fronts it in a real environment
   is ❓ unverified (see Open Questions).

## Consequences

**Positive**
- Minimal surface and zero auth configuration — the sample runs and is understandable with no
  identity plumbing.
- Clear, single hardening seam: a production deployment adds exactly one concern (authn/authz) at a
  known layer.

**Negative / trade-offs**
- ✅ **Every endpoint is publicly reachable** with no credentials, including the management surface
  (`/actuator/*`) and the H2 console — an information-disclosure and unauthenticated-management risk
  if exposed to an untrusted network (architecture-baseline §5).
- No audit trail of *who* performed an action (there is no authenticated principal).
- Any multi-user or multi-tenant requirement is unmet and would require a substantial addition.

**Related decision**
- The management/diagnostics **exposure posture** (Actuator `*`, H2 console) is a distinct, related
  decision tracked separately as candidate ADR-CANDIDATE-008 (second batch); this ADR covers only the
  absence of application authn/authz.

**Pattern relationship**
- No security-*authentication* pattern exists in the catalog to instantiate. The catalog's
  *Data-Binding Field Allowlist (Mass-Assignment Protection)* pattern
  (`docs/architecture-inventory/patterns/security/data-binding-field-allowlist.md`) is a
  **correctness/input-safety** control, not access control, and is orthogonal to this decision — so
  there is no **Pattern Drift**.
- **Pattern Update Proposal:** if authn/authz is later added, a new "authentication & authorization"
  pattern should be cataloged and this ADR superseded.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | No Spring Security (or equivalent) dependency | ✅ | `pom.xml`, `build.gradle` — `grep security` empty across both |
| E2 | No security filter chain / login / role checks | ✅ | `system/**`, controllers under `owner/`, `vet/`; architecture-baseline §5 |
| E3 | Actuator fully exposed, flagged dev-only | ✅ | `src/main/resources/application.properties:19–21` |
| E4 | Input-side safeguards are validation, not access control | ✅ | `owner/OwnerController.java` (`setAllowedFields`), `owner/PetValidator.java` (architecture-baseline §5) |
| E5 | CAKE auth/tenant ADRs govern a different platform | ✅ | `cake_graph_query` (2026-08-18): `ADR-MULTI-TENANT-DATA-ISOLATION`, `ADR-SESSION-BASED-AUTHENTICATION`, `ADR-006`, `ADR-PLATFORM-027` — absent from this tree |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The unauthenticated posture is intentional (reference/sample app), not an unfinished feature. | Well-known sample application; inline config comments flag dev-only exposure. | If real users were intended, the app is critically under-protected. | Confirm intended audience/deployment with the product/security owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a security decision owner accountable for the trust boundary. | Platform security team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | In real deployments, is the app fronted by an external gateway/ingress/network policy that enforces authentication and restricts the management surface? | If not, the fully-exposed, unauthenticated endpoints are a live security exposure. | ❓ Unverified in-repo; assumed dev-only per inline comments. | Security / platform owner |
| Q2 | If the platform is to serve real clinic users, which authn/authz model (Spring Security form vs OAuth2/OIDC) should be adopted? | Determines the hardening path and supersession of this ADR. | 🎯 OAuth2/OIDC if integrating an external IdP; otherwise Spring Security. | Security / architecture owner |
