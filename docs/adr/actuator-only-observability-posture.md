# ADR: Observability Posture — Actuator-Only, No Metrics Registry or Tracing

- **Status:** Accepted (retroactive — documents an existing, evidenced decision)
- **Date:** 2026-08-18
- **Deciders:** Platform architecture / operations team (owner attribution pending — see B1)
- **Candidate ID:** ADR-CANDIDATE-014 (from `docs/architecture-inventory/adr-candidates.md`)
- **Category:** Other (Observability) · **Impact:** medium · **Dependencies:** ADR-CANDIDATE-001

## Notation

| Symbol | Meaning |
|--------|---------|
| ✅ | Confirmed current behavior — cited to file path and line range |
| 🎯 | Target state — intended design, not yet in production |
| ❓ | Needs validation — assumed but not directly observed |

## Context

Observability for the single-process monolith (ADR-CANDIDATE-001) is limited to what Spring Boot
Actuator and JCache statistics provide out of the box; there is **no external metrics registry and
no distributed tracing**.

- ✅ Only `spring-boot-starter-actuator` is on the dependency path — Maven (`pom.xml:44`) and Gradle
  (`build.gradle:41`, `runtimeOnly`). There is **no** Micrometer registry dependency
  (`micrometer-registry-prometheus`/`-otlp`), **no** OpenTelemetry/`micrometer-tracing`, and **no**
  Zipkin dependency anywhere in `pom.xml` or `build.gradle` (grep for
  `micrometer|prometheus|opentelemetry|zipkin|otlp|tracing` returns only the Actuator starters).
- ✅ All Actuator endpoints are exposed (`management.endpoints.web.exposure.include=*`,
  `application.properties:21`) — so `/actuator/health`, `/actuator/metrics`, and `/actuator/sbom`
  are reachable, but `/actuator/metrics` serves only in-JVM Micrometer meters with no backend to
  scrape/store them. (Endpoint exposure risk is covered separately in
  [management-diagnostics-endpoint-exposure](management-diagnostics-endpoint-exposure.md).)
- ✅ Cache statistics are the one bespoke signal: `CacheConfiguration` enables JCache statistics
  accessible via JMX (`src/main/java/org/springframework/samples/petclinic/system/CacheConfiguration.java:28-29,41`)
  for the `vets` cache (`vet/VetRepository.java:45,55`) — see
  [in-process-read-through-cache](in-process-read-through-cache.md).
- ✅ Kubernetes consumes only the health probes (`/livez`, `/readyz`, `k8s/petclinic.yml:47-54`);
  no scrape annotations, ServiceMonitor, or OTLP exporter config exist in `k8s/*.yml`.

For a single in-JVM process this is a defensible baseline — health checks drive orchestration and
JMX/JCache stats give local cache visibility — but there is no cross-process telemetry, no metrics
time-series retention, and no request tracing.

## Decision

**Keep observability limited to Spring Boot Actuator (health probes, in-JVM metrics endpoint, SBOM)
plus JCache/JMX cache statistics, and do NOT add a metrics-registry backend (Prometheus/OTLP) or
distributed tracing (OpenTelemetry/Zipkin) at this time.** Health endpoints remain the
orchestrator-facing contract; metrics and tracing backends are deferred until a concrete
operational need (production SLOs, multi-instance latency debugging) justifies the added
dependencies and infrastructure.

## Alternatives Considered

1. **Add a Micrometer metrics registry (e.g. Prometheus) with a scrape/exporter pipeline.**
   *Rejected for now (🎯 target once in production)* — a Prometheus registry plus scrape
   configuration would give retained time-series, dashboards, and alerting the current setup lacks.
   It was not adopted because it requires a new dependency, exporter/scrape wiring in `k8s/*.yml`,
   and a monitoring backend to run — infrastructure that a single-replica demo deployment
   (`k8s/petclinic.yml:22`) does not have. This is the first thing to add when the platform runs
   production traffic (Q1).

2. **Add distributed tracing (OpenTelemetry / `micrometer-tracing` + Zipkin/OTLP).**
   *Rejected as low-value for a monolith today* — tracing's payoff is following a request across
   process/service boundaries. With a single deployable and no downstream service calls
   (ADR-CANDIDATE-001), there is no cross-service span graph to reconstruct, so tracing would add
   dependencies and an OTLP collector for little marginal insight. It becomes valuable only if the
   monolith is decomposed or gains outbound integrations.

3. **No observability beyond default health checks (drop even JCache stats/JMX).**
   *Rejected* — this would remove the cache-hit/miss visibility that `CacheConfiguration` explicitly
   enables (`CacheConfiguration.java:28-29`), which is the one signal that lets the read-through
   cache decision be validated operationally. Keeping Actuator + JCache stats is a strictly better
   baseline at no added cost.

## Consequences

**Positive**
- ✅ Zero observability infrastructure to run: health probes drive Kubernetes
  (`k8s/petclinic.yml:47-54`) and cache stats are available via JMX with no external backend
  (`CacheConfiguration.java:28-29`).
- ✅ The Actuator surface (health, metrics, SBOM) is present and can be scraped later by simply
  adding a registry dependency — the metrics endpoint already exists (`application.properties:21`).

**Negative / trade-offs**
- ✅ **No retained metrics:** `/actuator/metrics` exposes live in-JVM meters only; without a registry
  there is no time-series storage, dashboarding, or alerting. Production operability is limited.
- ✅ **No distributed tracing:** request latency cannot be decomposed; multi-instance behavior
  (each replica holds its own cache — [in-process-read-through-cache](in-process-read-through-cache.md))
  cannot be correlated.
- ❓ There is no evidence of log aggregation/shipping either — logging is console at `INFO`
  (`application.properties:24`); whether logs are collected in the cluster is not evidenced (Q2).

**Pattern relationship**
- **Instantiates** the *Health Probe & Management-Endpoint Exposure* pattern
  (`patterns/observability/health-probe-and-management-endpoints.md`, Candidate) — the health-probe
  half is realized; the metrics/tracing half is deliberately deferred. **No Pattern Drift.**
- **Pattern Update Proposal:** extend the observability pattern with a "metrics/tracing readiness"
  note describing the minimal registry/exporter wiring to add when the platform is production-bound,
  so the upgrade path is consistent.

## Evidence

| # | Claim | Notation | Evidence |
|---|-------|----------|----------|
| E1 | Only Actuator is on the dependency path (no Micrometer registry/tracing/Zipkin) | ✅ | `pom.xml:44`; `build.gradle:41`; absence of `micrometer-registry-*`/`opentelemetry`/`zipkin` deps |
| E2 | All Actuator endpoints exposed, including metrics/SBOM | ✅ | `src/main/resources/application.properties:18-21` |
| E3 | JCache statistics enabled and exposed via JMX for the `vets` cache | ✅ | `system/CacheConfiguration.java:28-29,41`; `vet/VetRepository.java:45,55` |
| E4 | Kubernetes consumes health probes only; no scrape/OTLP config | ✅ | `k8s/petclinic.yml:47-54`; absence of scrape annotations/ServiceMonitor in `k8s/*.yml` |
| E5 | Logging is console-level `INFO`, no aggregation configured | ✅ | `application.properties:24` |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts.

### Assumptions
| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The Actuator-only posture is intentional for a demo/reference deployment, not an oversight. | Consistent with the single-replica demo topology and the dev-only endpoint-exposure comments (`application.properties:19-20`). | If production operability was expected, the missing registry/tracing is a gap, not a decision. | Confirm production observability requirements with the operations owner. |
| A2 | No out-of-repo metrics/tracing sidecar or agent is injected at deploy time. | `k8s/*.yml` contains no exporter/collector/agent config. | If a cluster-side agent scrapes/traces the app, the posture is richer than in-repo evidence shows. | Confirm cluster observability tooling with the platform operations owner. |

### Blockers
| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | No CAKE repo→service / owning-team record exists for this clone. | Assigning a decision owner to this ADR and to Q1/Q2. | Platform architecture / operations team | Register the repo→service mapping and owning team in CAKE, or supply it out-of-band. | TBD |

### Open Questions
| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | Under what condition is a metrics registry (Prometheus/OTLP) added? | Determines the trigger for moving from demo observability to production SLO monitoring. | 🎯 Add a Micrometer registry + scrape/export wiring when the platform serves production traffic or requires alerting. | Platform operations owner |
| Q2 | Are application logs aggregated/shipped in the cluster, or only written to stdout? | Without aggregation there is no post-hoc diagnosis across replicas/restarts. | — | Platform operations owner |
