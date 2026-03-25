# Poly Labs Central

**GitHub**: [polylabs-dev/polylabs](https://github.com/polylabs-dev/polylabs)
**Platform**: eStream v0.22.0
**Depends on**: PolyKit v0.4.0, eStream graph/DAG constructs

100% FastLang. No hand-written Rust.

## Purpose

Central monorepo for Poly Labs: backend services (gateway, billing, enterprise admin), and the unified console. All logic lives in FastLang circuits compiled via FLIR codegen.

> **Note**: `crates/`, `sdks/`, and `console/` directories are legacy scaffolding superseded by FLIR codegen. All backend services, SDKs, and console functionality are now produced by FL circuit compilation.

## Zero-Linkage Privacy

This repo enforces the zero-linkage privacy architecture. The billing system uses **blinded payment tokens** derived from a separate HKDF context (`billing-v1`). No backend system in this repo can correlate user identities across products. Enterprise customers can opt-in to cross-product org-level visibility via a lex bridge gated by k-of-n admin witness attestation.

## Structure

- `circuits/` — FastLang circuit definitions (v0.22.0 pattern)
  - `gateway/` — Request routing, SPARK auth, global rate limiting (3 circuits)
  - `billing/` — Subscription FSM, 8D metering, blinded payments, L2 settlement (4 circuits)
  - `admin/` — Platform RBAC graph, audit trail, product provisioning (3 circuits)
  - `graphs/` — Tenant registry graph, billing history DAG (2 circuits)
- `marketplace/` — eStream Marketplace component manifests
- `docs/` — Architecture and design documents

## Circuits (12 total)

| Domain | Circuit | Key Pattern |
|--------|---------|-------------|
| Gateway | `gateway_router` | Wire protocol demux, `@guard(block_on: unknown_product)` |
| Gateway | `gateway_auth` | SPARK auth, `constant_time`, `fuzz_target` |
| Gateway | `gateway_rate_limit` | Two-tier rate limiting, `@guard(block_on: global_rate_exceeded)` |
| Billing | `billing_subscription` | `state_machine subscription_tier` (Free/Premium/Pro/Enterprise/Cancelled) |
| Billing | `billing_metering` | 8D metering aggregation, composes PolyKit metering |
| Billing | `billing_blinded` | Blinded payment verification, `constant_time`, `no_identity_leak` safety |
| Billing | `billing_settlement` | `state_machine settlement_fsm`, L2 multi-token (USDC/SOL/cbBTC) |
| Admin | `admin_rbac` | `graph admin_role_graph`, 4 platform roles, CSR storage |
| Admin | `admin_audit` | ML-DSA-87 signed hash chain, PoVC attestation |
| Admin | `admin_provisioning` | Lex namespace creation, scatter node allocation |
| Graphs | `tenant_graph` | `graph tenant_registry` (Customer/Subscription/Product), AI feeds |
| Graphs | `billing_dag` | `dag billing_history` (Invoice/Payment/Credit), merkle chain |

## Commit Convention

Commit to the GitHub issue or epic the work was done under. Do not accumulate large amounts of uncommitted work.

## v0.22.0 Circuit Pattern

Every data declaration follows:
```
data X : namespace v1 {
    field: type,
}
    store kv|graph
    govern lex esn/global/org/polylabs/...
    cortex { redact [...], infer on_write, on_anomaly alert "team" }
```

Every circuit must have: `lex`, `precision`, `observe metrics`, at least 1 `invariant`, at least 1 `property` (safety or liveness), at least 1 `test golden`.

## Platform

- eStream v0.22.0
- PolyKit v0.4.0
- FLIR codegen (FastLang → FLIR → Rust/WASM)
- ML-KEM-1024, ML-DSA-87, SHA3-256
- 8-Dimension metering
- Blinded billing tokens
