# PolyLabs Backend — CE + App Graph Spec

> **Version**: 0.1.0
> **Status**: Active
> **Platform**: eStream v0.22.0
> **Lex Root**: `polyqlabs/backend`

---

## Overview

The PolyLabs backend exposes 12 FastLang circuits across four subsystems (gateway, billing, admin, graphs). This spec covers two integration layers:

1. **App Graph** — registers the 12 modules into the Stratum module graph with typed dependency edges, cross-graph bridges to QKit and eStream primitives, and governance observation edges.
2. **Cognitive Engine** — defines 3 meaning domains, 1 noise filter configuration, and 2 SME panels so the backend feeds operational intelligence into the CE pipeline.

### Design Goals

| Goal | Mechanism |
|------|-----------|
| Unified module registry | All 12 modules registered in a single `CsrStorage` graph via `module_graph_add_module` |
| Typed dependency tracking | 18 `EDGE_REQUIRES` edges encode the backend's internal dependency DAG |
| Cross-graph composability | 4 `EDGE_BRIDGE_TO` edges to QKit metering, RBAC, SPARK, and scatter-cas |
| Governance observability | 12 `EDGE_GOVERNANCE_OBSERVE` edges — one per module |
| Per-domain CE isolation | Each meaning domain scoped under `polyqlabs/backend/cognitive` lex |
| Noise suppression | Health checks, internal probes, and synthetic traffic filtered before CE ingestion |

---

## App Graph — 12 Modules

### Module Inventory

| Group | Module | Partition | SLA | Role |
|-------|--------|-----------|-----|------|
| **Gateway** | `gateway_router` | Head | Premium | Wire protocol demux, product-aware routing |
| | `gateway_auth` | Head | Premium | SPARK auth verification, constant-time |
| | `gateway_rate_limit` | Head | Premium | Two-tier rate limiting (global + per-product) |
| **Billing** | `billing_subscription` | Backend | Standard | Tier FSM: Free/Premium/Pro/Enterprise/Cancelled |
| | `billing_metering` | Backend | Standard | 8D metering aggregation via QKit |
| | `billing_blinded` | Backend | Premium | Blinded payment verification, no identity leak |
| | `billing_settlement` | Backend | Premium | L2 settlement FSM, multi-token |
| **Admin** | `admin_rbac` | Backend | Standard | Platform RBAC graph, 4 roles |
| | `admin_audit` | Backend | Standard | ML-DSA-87 signed audit hash chain |
| | `admin_provisioning` | Backend | Standard | Lex namespace creation, scatter node allocation |
| **Graphs** | `tenant_graph` | Backend | Standard | Customer/Subscription/Product relationship registry |
| | `billing_dag` | Backend | Standard | Invoice/Payment/Credit history DAG |

### Dependency Edges (18 EDGE_REQUIRES)

```
gateway_router → gateway_auth, gateway_rate_limit, tenant_graph
gateway_auth → admin_rbac
gateway_rate_limit → billing_metering
billing_subscription → tenant_graph
billing_metering → billing_subscription, tenant_graph
billing_blinded → billing_subscription
billing_settlement → billing_blinded, billing_dag
billing_dag → billing_metering
admin_audit → admin_rbac, billing_settlement
admin_provisioning → tenant_graph, admin_rbac, admin_audit
tenant_graph → billing_subscription
```

### Bridge Edges (4 EDGE_BRIDGE_TO)

| Source Module | Target Lex | Target Module | Bridge Type |
|---------------|-----------|---------------|-------------|
| `billing_metering` | `qkit/metering` | `qkit_metering` | metering_aggregation |
| `admin_rbac` | `qkit/rbac` | `qkit_rbac` | role_composition |
| `gateway_auth` | `core/identity/spark` | `spark_verify` | auth_verification |
| `billing_blinded` | `core/storage/scatter` | `scatter_cas` | blinded_token_storage |

---

## Cognitive Engine — 3 Domains, 1 Filter, 2 Panels

### Meaning Domains

| Domain Path | Description | Crystallization | Impact |
|-------------|-------------|-----------------|--------|
| `platform/gateway_health` | Auth failure patterns, rate limit triggers, latency anomalies, error rate spikes | 70 | 80 |
| `platform/billing_patterns` | Churn signals, tier migration velocity, usage spikes, payment failures | 80 | 90 |
| `platform/tenant_lifecycle` | Provisioning velocity, lex utilization trends, inactive tenant detection | 75 | 70 |

### Noise Filter

| Config | Value |
|--------|-------|
| Suppress health checks | `true` (kube-probe, ELB-HealthChecker, GoogleHC) |
| Suppress internal probes | `true` (10.0.*, 172.16.*) |
| Suppress synthetic traffic | `true` |
| Min signal confidence | 60 |
| Dedup window | 300,000 ms |

### SME Panels

| Panel | Domain Scope | Min Panelists | Specializations | Calibration Floor |
|-------|-------------|---------------|-----------------|-------------------|
| Gateway Optimization | `platform/gateway_health` | 3 | auth_security, rate_limiting, routing_optimization, latency_analysis | 75.00% |
| Billing Evolution | `platform/billing_patterns` | 3 | churn_prediction, pricing_optimization, tier_migration, payment_processing | 80.00% |

---

## Circuit Files

| File | Lines | Contents |
|------|-------|----------|
| `circuits/fl/polyqlabs_app_graph.fl` | ~250 | 12 module definitions, graph registration, bridge edges, governance edges, properties, golden tests |
| `circuits/fl/polyqlabs_meaning.fl` | ~150 | 3 domain data types, noise filter config, 2 SME panel types, registration circuits, golden tests |

---

## Verification

| Property | Type | Assertion |
|----------|------|-----------|
| `all_modules_registered` | Safety | All 12 modules findable by name after registration |
| `graph_registration_completes` | Liveness | `num_nodes >= 12` after `polyqlabs_app_graph_register` |
| `register_all_12_modules` | Golden test | Node count = 12, spot-check 3 modules |
| `bridge_edges_to_qkit` | Golden test | Bridge registration increases node/edge count |
| `governance_edges_all_modules` | Golden test | Governance registration adds >= 12 edges |
| `full_ce_pipeline_setup` | Golden test | All 3 domains + filter + 2 panels initialize |

---

## References

- eStream MCP App Graph: `estream/circuits/core/intelligence/mcp/mcp_app_graph.fl`
- QKit CE Spec: `qkit/specs/POLYKIT_CE_APP_GRAPH_SPEC.md`
- QKit Cognitive: `qkit/circuits/fl/qkit_cognitive.fl`
- QKit Noise Filter: `qkit/circuits/fl/qkit_noise_filter.fl`
- QKit SME: `qkit/circuits/fl/qkit_sme.fl`
