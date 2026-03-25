# Poly Labs Central Architecture

**Version**: 2.0.0
**Date**: March 2026
**Platform**: eStream v0.22.0
**Upstream**: PolyKit v0.5.0, eStream graph/DAG constructs, FLIR codegen

---

## Overview

The `polylabs` repo is the **central monorepo** for Poly Labs. It contains:

- **FL backend circuits** — gateway, billing, admin, tenant management (12 circuits, pure FL)
- **PolySDKs** — FLIR-codegen'd embeddable SDKs, published to eStream Marketplace
- **Console** — unified management UI composing PolyKit React components with codegen'd widget data
- **Marketplace manifests** — eStream Marketplace component definitions

All 37 products across 7 verticals are independent repos. This repo provides the **cross-cutting infrastructure** that connects them — without violating zero-linkage privacy.

### FL-Native Stack

```
┌─────────────────────────────────────────────────────────────────┐
│  Product Layer — 37 products, pure FL                           │
│  (polydocs, polymessenger, polymail, polygit, polyfiles, ...)   │
│  Each product: ~8-15 FL circuits                                │
├─────────────────────────────────────────────────────────────────┤
│  PolyKit — 30 shared circuits                                   │
│  (user_graph, metering, subscription, lex_bridge, RBAC profiles,│
│   React bindings, UI primitives, theme, accessibility)          │
├─────────────────────────────────────────────────────────────────┤
│  Poly Labs Central — 12 circuits (this repo)                    │
│  (gateway, billing, admin, tenant, SDK codegen, marketplace)    │
├─────────────────────────────────────────────────────────────────┤
│  eStream v0.22.0 — Self-hosting platform                        │
│  (FastLang compiler, FLIR, scatter-cas, es-git, SPARK, lattice, │
│   StreamSight, ML-DSA-87/ML-KEM-1024, 17 codegen backends)     │
└─────────────────────────────────────────────────────────────────┘
```

No hand-written Rust exists in any product repo. FL circuits compile via FLIR codegen to WASM (browser/Tauri) and Rust (server), producing deployable artifacts directly. The only Rust in the entire stack is eStream's ~127 LOC OS I/O boundary.

---

## FL-to-WASM Paradigm Shift

### Before (v0.11.0 era)

The original architecture relied on hand-written Rust crates for backend services and manually maintained SDK packages:

- `crates/polylabs-gateway/` — Rust gateway binary
- `crates/polylabs-billing/` — Rust billing service
- `crates/polylabs-admin/` — Rust admin service
- `sdks/polysdk-auth/` — hand-written React + WASM bindings
- `sdks/polysdk-messenger/` — hand-written React + WASM bindings

This created dual maintenance: FL circuits defined the logic, but Rust crates duplicated it for deployment.

### After (v0.22.0 — current)

FLIR codegen eliminates all hand-written Rust. The FL circuit **is** the implementation:

```
polylabs_gateway.fl  ──FLIR──▶  polylabs_gateway.wasm  (Cloudflare Worker)
                     ──FLIR──▶  polylabs_gateway.rs    (lattice node)

polylabs_billing.fl  ──FLIR──▶  polylabs_billing.wasm  (browser metering)
                     ──FLIR──▶  polylabs_billing.rs    (settlement node)
```

Benefits:
- **Single source of truth**: The `.fl` file is both specification and implementation
- **No Rust maintenance**: Codegen'd Rust is deterministic, auditable, never hand-edited
- **Multi-target**: Same circuit deploys to WASM (browser), Rust (server), or Verilog (FPGA) via FLIR backend selection
- **Type-safe SDKs**: SDK interfaces are extracted from FL circuit signatures, not hand-written

---

## Zero-Linkage Privacy Architecture

### The Problem

A privacy-focused product suite creates a tension: users want a unified experience, but cross-product linkage creates a subpoena surface. If the backend can prove that user X uses both Poly Messenger and Poly Files, then a legal request for Messenger data implicitly reveals the existence of their Files.

### The Solution: Blinded Product Isolation

```
                    ┌─────────────────────────────────┐
                    │     SPARK Biometric (device)     │
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────┴────────────────────┐
                    │     HKDF-SHA3-256 (in WASM)     │
                    └──┬──────┬──────┬──────┬─────────┘
                       │      │      │      │
              ┌────────┴┐ ┌───┴────┐ │  ┌───┴────────┐
              │files-v1 │ │msg-v1  │ │  │mail-v1     │
              │user_id_A│ │user_id_B│ │  │user_id_C   │
              └─────────┘ └────────┘ │  └────────────┘
                                     │
                            ┌────────┴────────┐
                            │billing-v1       │
                            │blinded_token    │
                            └─────────────────┘
```

- Each of the 37 products derives an independent `user_id` from a product-specific HKDF context
- The billing system receives a blinded payment token, not a SPARK identity
- No backend system can correlate user_id_A (Poly Files) with user_id_B (Poly Messenger)
- StreamSight telemetry is per-product lex namespace; no cross-product aggregation

### Billing Token Flow

```
User -> SPARK biometric -> WASM derives billing_token via HKDF("billing-v1")
     -> billing_token is blinded (randomized commitment)
     -> Payment processor sees: blinded_token + tier + amount
     -> Payment processor does NOT see: which products, which user_id, SPARK identity
     -> Each product checks tier via its own lex: "is this user's tier >= PREMIUM?"
     -> Tier status is broadcast per-product (product cannot see other products' tiers)
```

### Individual Opt-In Bridge

Individual users can choose to link their own products for a unified personal dashboard. This is entirely client-side — the backend never learns about the linkage unless the user publishes it.

```fastlang
lex_bridge polylabs_personal {
    scope personal
    owner spark_user_id

    source esn/global/org/polylabs/files/{user_id_A}
    source esn/global/org/polylabs/messenger/{user_id_B}
    source esn/global/org/polylabs/mail/{user_id_C}
    target esn/global/org/polylabs/personal/{bridge_id}

    sign ml_dsa_87
    signer_context "poly-bridge-v1"

    allowed_fields [storage_used, message_count, unread_count, tier, device_count]
    denied_fields [file_content, message_content, contact_list, encryption_keys]

    revocable true
    audit_stream local_only
}
```

The bridge is **client-side only** (WASM + ESLite), **content-free** (aggregates only), **cryptographically bound** to the SPARK master seed, and **revocable** by deleting the bridge key. No server or lattice node learns about the linkage — zero subpoena surface.

### Enterprise Opt-In Bridge

Enterprise admins can choose to enable cross-product visibility at the org level:

```fastlang
lex_bridge polylabs_enterprise {
    scope organization
    owner org_admin

    source esn/global/org/polylabs/files
    source esn/global/org/polylabs/messenger
    target esn/global/org/polylabs/admin

    witness_attest true
    witness_k 3
    witness_n 5

    allowed_fields [org_id, seat_count, storage_aggregate, compliance_status]
    denied_fields [user_id, file_id, message_id, content]

    revocable true
    audit_stream esn/global/org/polylabs/admin/bridge_audit
}
```

The enterprise bridge requires k-of-n admin witness attestation (3-of-5), passes only org-level aggregates, explicitly denies user-level identifiers, and is revocable and audited via tamper-proof series.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Poly Labs Central (polylabs/)                                          │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Console (PolyKit React + codegen'd widget data)                  │  │
│  │  ├── Consumer Dashboard    subscription, usage, product links     │  │
│  │  ├── Enterprise Admin      org mgmt, RBAC, compliance, audit     │  │
│  │  └── Widget Registry       codegen'd from FL circuit signatures   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  PolySDKs (FLIR codegen'd from FL circuit signatures)             │  │
│  │  ├── @polysdk/auth         SPARK biometric auth component         │  │
│  │  ├── @polysdk/messenger    embeddable messaging                   │  │
│  │  ├── @polysdk/files        embeddable file browser/storage        │  │
│  │  ├── @polysdk/suite        all-in-one bundle                      │  │
│  │  └── ... (per-product SDKs generated on demand)                   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  FL Circuits (12 circuits — compiled via FLIR to WASM + Rust)     │  │
│  │  ├── polylabs_gateway.fl          central gateway + routing       │  │
│  │  ├── polylabs_billing.fl          blinded token subscription      │  │
│  │  ├── polylabs_admin.fl            enterprise admin                │  │
│  │  ├── polylabs_tenant.fl           multi-tenant isolation          │  │
│  │  ├── polylabs_sdk_codegen.fl      SDK generation from signatures  │  │
│  │  ├── polylabs_marketplace.fl      component publishing            │  │
│  │  ├── polylabs_metering.fl         usage metering (per-product)    │  │
│  │  ├── polylabs_bridge.fl           opt-in lex bridge management    │  │
│  │  ├── polylabs_onboarding.fl       product provisioning            │  │
│  │  ├── polylabs_compliance.fl       cross-product compliance        │  │
│  │  ├── polylabs_notification.fl     notification routing            │  │
│  │  └── polylabs_analytics.fl        org-level analytics (bridged)   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  Composes: PolyKit v0.5.0 + eStream group_hierarchy.fl + rbac.fl      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7-Vertical Product Family (37 Products)

Each product is a thin composition of ~8-15 FL circuits over PolyKit and eStream primitives. No hand-written Rust in any product repo.

### Vertical 1: Productivity (7 products, ~70 circuits)

| Product | Circuits | Key FL Constructs |
|---------|----------|-------------------|
| Poly Docs | ~12 | Document Engine (125+ FL), CRDT sync, scatter-cas |
| Poly Slides | ~10 | Document Engine, presentation renderer |
| Poly Sheets | ~10 | Document Engine, cell engine, formula parser |
| Poly Notes | ~8 | Markdown parser, rich text, scatter-cas |
| Poly Calendar | ~10 | Event graph, scheduling DAG, invite circuit |
| Poly Tasks | ~10 | Task graph, state machines, kanban overlay |
| Poly Forms | ~10 | Form schema graph, submission DAG, compliance |

### Vertical 2: Communication (3 products, ~35 circuits)

| Product | Circuits | Key FL Constructs |
|---------|----------|-------------------|
| Poly Messenger | ~15 | contact_network graph, message_thread DAG, relay_mesh, SFU |
| Poly Mail | ~12 | mailbox_registry graph, email_thread DAG, SMTP/IMAP bridge |
| Poly Phone | ~8 | VoIP circuit, SRTP, TURN relay, cellular fallback |

### Vertical 3: Security & Identity (4 products, ~40 circuits)

| Product | Circuits | Key FL Constructs |
|---------|----------|-------------------|
| Poly Pass | ~10 | vault_registry graph, credential_lifecycle SM |
| Poly OAuth | ~12 | identity_federation graph, token_chain DAG, ML-DSA-87 PQ-JWT |
| Poly VPN | ~10 | vpn_exit_mesh graph, tunnel_route DAG, traffic mimicry |
| Poly Authenticator | ~8 | TOTP/HOTP circuits, FIDO2/WebAuthn attestation |

### Vertical 4: Developer Tools (5 products, ~55 circuits)

| Product | Circuits | Key FL Constructs |
|---------|----------|-------------------|
| Poly Git | ~15 | es-git, commit_graph, governance DAG, CI/CD engine |
| Poly Studio | ~15 | Studio IDE (34 FL), 12-layer overlay, LSP bridge |
| Poly CI | ~8 | CI job circuit, witness-attested pipelines |
| Poly Registry | ~10 | Package graph, scatter-cas, ML-DSA-87 signing |
| Poly Monitor | ~10 | StreamSight composition, APM graph, alert DAG |

### Vertical 5: Enterprise & Compliance (5 products, ~50 circuits)

| Product | Circuits | Key FL Constructs |
|---------|----------|-------------------|
| Poly Comply | ~12 | Compliance Engine (18 FL), multi-framework automation |
| Poly Sign | ~10 | ML-DSA-87 signing, witness attestation, audit trail |
| Poly Archive | ~10 | Archival graph, legal hold SM, cold tier scatter-cas |
| Poly Audit | ~8 | Audit series, merkle_chain, tamper detection |
| Poly Vault | ~10 | HSM bridge, PRIME key management, threshold signing |

### Vertical 6: Consumer & Media (4 products, ~40 circuits)

| Product | Circuits | Key FL Constructs |
|---------|----------|-------------------|
| Poly Photos | ~12 | Media Engine (11 FL), provenance graph, scatter-cas |
| Poly Contacts | ~8 | Contact graph, SPARK linking, import/export |
| Poly Stream | ~10 | ABR streaming, Media Engine, DRM circuit |
| Poly Mind | ~10 | knowledge_corpus graph, ESLM, legacy_governance DAG |

### Vertical 7: Hardware-Enhanced (7 products, ~65 circuits)

Software-first MVPs, with Companion SoC (T0) enhancement in later phases.

| Product | Circuits | Key FL Constructs |
|---------|----------|-------------------|
| Poly Files | ~12 | Scatter-cas, file_registry graph, share_network graph |
| Poly Wallet | ~10 | Multi-asset circuit, threshold signing, NFC bridge |
| Poly Camera | ~8 | SPARK attestation, provenance, Media Engine |
| Poly Find | ~8 | Location graph, PQ-encrypted sharing, geofence |
| Poly Health | ~10 | Biometric graph, delta_curate, TEE attestation |
| Poly Browse | ~10 | PQ-DNS, ad blocking, content filter DAG |
| Poly Maps | ~7 | Offline tile graph, route DAG, PQ-encrypted POI |

**Totals**: ~355 product-level circuits across 37 products, plus 30 PolyKit circuits and 12 Poly Labs Central circuits.

---

## Graph/DAG Constructs

### Subscription Graph (`polylabs_billing.fl`)

Manages subscriptions using blinded tokens. The graph does NOT store SPARK identities or product-specific user_ids. It operates on anonymous payment tokens.

```fastlang
graph subscription_graph {
    node BlindedAccountNode    // token_hash, tier, created_at, billing_cycle_start
    node ProductEntitlementNode // entitlement_id, product_code, tier, valid_from/until
    edge SubscribesEdge        // subscribed_at, payment_method_hash

    overlay tier: u8 curate delta_curate
    overlay mrr_cents: u64 bitmask delta_curate
    overlay churn_risk: u32 bitmask delta_curate

    storage csr { hot @bram, warm @ddr, cold @nvme }
    ai_feed churn_prediction

    observe subscription_graph: [tier, churn_risk, mrr_cents] threshold: {
        anomaly_score 0.8
        baseline_window 300
    }
}

series subscription_series: subscription_graph
    merkle_chain true
    lattice_imprint true
    witness_attest true
```

Key constraint: `BlindedAccountNode.token_hash` is derived from `HKDF("billing-v1")` — a separate derivation context that cannot be correlated to any product-specific identity.

### Enterprise Org Graph (composes `group_hierarchy.fl`)

For enterprise customers who opt in, the `polylabs_admin.fl` circuit extends eStream's `group_hierarchy.fl`:

```fastlang
type PolyOrgExtension = struct {
    org_id: bytes(32),
    licensed_products: u64,   // expanded from u16 for 37-product family
    seat_limit: u32,
    contract_tier: u8,
    contract_expires: u64,
}

// Key circuits (3 of ~5 in polylabs_admin.fl)
circuit assign_product_license(hierarchy: bytes(32), org_id: bytes(32), product_code: u8) -> bool
    lex esn/global/org/polylabs/admin  precision C  rbac [org_admin]  povc true

circuit enforce_seat_limit(hierarchy: bytes(32), org_id: bytes(32)) -> bool
    lex esn/global/org/polylabs/admin  precision B

circuit org_compliance_report(hierarchy: bytes(32), org_id: bytes(32)) -> bytes
    lex esn/global/org/polylabs/admin/compliance  precision B  rbac [compliance_admin]
```

---

## PolySDK Architecture

PolySDKs are **no longer hand-written**. They are generated from FL circuit signatures via FLIR's `service_dispatch` and `tool_registry` codegen backends.

### How SDK Codegen Works

```
product_circuit.fl
  │
  ├── FLIR analysis ──▶ Extract public circuit signatures
  │                      (inputs, outputs, lex paths, RBAC requirements)
  │
  ├── service_dispatch ──▶ Generate typed service client
  │                        (TypeScript types, WASM bindings, error types)
  │
  └── tool_registry ──▶ Generate React hooks + components
                         (usePolyMessenger, <PolyMessenger />, etc.)
```

### Generated SDK Structure

For each product, FLIR codegen produces:

```
@polysdk/{product}/
├── dist/
│   ├── polysdk-{product}.wasm    ← FLIR WASM backend output
│   ├── index.js                   ← Generated React bindings
│   ├── index.d.ts                 ← Generated TypeScript types
│   └── hooks.js                   ← Generated React hooks
├── estream-component.toml         ← Marketplace manifest
└── package.json                   ← npm package (auto-versioned)
```

### SDK Usage (unchanged API surface)

The generated SDKs maintain the same developer-facing API:

```tsx
import { SparkAuth, useSparkIdentity } from '@polysdk/auth';
import { PolyMessenger, usePolyMessage } from '@polysdk/messenger';
import { PolyFileBrowser, usePolyFile } from '@polysdk/files';
import { PolySuiteProvider } from '@polysdk/suite';

// Auth
<SparkAuth onAuthenticated={handleAuth} />

// Messaging
<PolyMessenger contactId={sparkDid} />

// Files
<PolyFileBrowser path="/documents" classification="CONFIDENTIAL" />

// Suite bundle
<PolySuiteProvider products={['messenger', 'files', 'auth']} hkdfContext="partner-app-v1">
  <App />
</PolySuiteProvider>
```

The difference is that these components and hooks are **generated** from the FL circuit's public interface, not hand-maintained. When a circuit's signature changes, the SDK updates automatically on next codegen pass.

### Marketplace Packaging

Each generated SDK is published to eStream Marketplace:

```toml
[component]
name = "@polysdk/messenger"
version = "1.0.0"
category = "sdk"
description = "Embeddable PQ-encrypted messaging — FLIR codegen'd from polymessenger circuits"
license = "Commercial"
publisher = "polylabs"
codegen = "flir-service-dispatch"

[component.targets]
wasm = true
fpga = false

[component.dependencies]
polykit = "^0.5.0"

[component.sdk]
entry_wasm = "dist/polysdk-messenger.wasm"
entry_react = "dist/index.js"
framework = "react"
supports_react_native = true
source_circuits = ["polymessenger/circuits/fl/*.fl"]
```

---

## Console

The Poly Labs console is the unified management interface. It composes **PolyKit React components** with **codegen'd widget data** from the Poly Labs Central FL circuits.

### How Console Widgets Work

```
polylabs_billing.fl  ──FLIR──▶  Widget data types + hooks
polykit/react/       ──────────▶  Layout components, theme, accessibility
console/src/         ──────────▶  Page composition (thin glue layer)
```

Each console widget is a PolyKit `<DataWidget>` component bound to a codegen'd data hook:

```tsx
// Generated by FLIR from polylabs_billing.fl circuit signatures
const { subscription, tier, usage } = useSubscriptionData();

// PolyKit React component with codegen'd props
<DataWidget
  source={subscription}
  layout="card"
  theme={polyTheme}
  fields={['tier', 'billing_cycle', 'products']}
/>
```

### Consumer Dashboard

- Product launcher (links to individual product UIs across all 37 products)
- Subscription management (tier, payment method — all via blinded tokens)
- Usage overview (per-product metering, no cross-product correlation)
- Account settings (SPARK devices, guardians, recovery)

### Enterprise Admin (opt-in bridge required)

- Org management (via eStream `group_hierarchy.fl`)
- RBAC management (via eStream `rbac.fl`)
- Seat management and product licensing (37 products)
- Compliance reporting (GDPR, FedRAMP, CMMC, PCI, SOX, NIST, HIPAA)
- Cross-product usage aggregates (org-level only, never user-level)

### Console Widgets

| Widget | Audience | Data Source |
|--------|----------|-------------|
| `polylabs-subscription` | Consumer | `subscription_graph` (codegen'd) |
| `polylabs-usage-summary` | Consumer | Per-product metering (isolated, codegen'd) |
| `polylabs-device-manager` | Consumer | Per-product `user_graph` (isolated) |
| `polylabs-product-launcher` | Consumer | Product registry (37 products) |
| `polylabs-org-hierarchy` | Enterprise | `group_hierarchy.fl` (bridged) |
| `polylabs-rbac-manager` | Enterprise | `rbac.fl` (bridged) |
| `polylabs-seat-manager` | Enterprise | `polylabs_admin.fl` (codegen'd) |
| `polylabs-compliance` | Enterprise | Bridged audit streams |
| `polylabs-analytics` | Enterprise | `polylabs_analytics.fl` (org-level) |

---

## eStream Upstream Compositions

Rather than reinventing core platform constructs, Poly Labs composes these eStream v0.22.0 production `.fl` files:

| eStream Construct | File | Poly Labs Usage |
|-------------------|------|-----------------|
| `graph rbac` | `rbac.fl` | PolyKit wraps `check_permission()` and `resolve_permissions()` into profiles. Per-product RBAC instances (zero-linkage). |
| `graph group_hierarchy` | `group_hierarchy.fl` | Enterprise org structure (`OrgNode`, `GroupNode`, `RepoNode`). |
| `state_machine issue_lifecycle` | `issue_tracking.fl` | Lifecycle pattern reused across all products for entity state machines. |
| Document Engine | 125+ FL circuits | Productivity vertical (Docs, Slides, Sheets, Notes, Forms). |
| Media Engine | 11 FL circuits | Consumer & Media vertical (Photos, Stream, Camera). |
| Compliance Engine | 18 FL circuits | Enterprise vertical (Comply, Audit, Archive). |
| CI/CD Engine | 4+ FL circuits | Developer Tools vertical (Git, CI). |
| Studio IDE | 34 FL circuits | Developer Tools vertical (Studio). |
| UI Framework | 9 FL circuits | PolyKit React bindings, widget system. |
| FLIR | 17 codegen backends | All SDK generation, all deployment artifacts. |
| SPARK | Biometric identity | Per-product isolated HKDF contexts (37 products = 37 independent identities). |
| StreamSight | Observability | Per-product isolated lex namespaces. |
| ESLM | On-device AI | Poly Mind, Cortex AI integration. |

---

## Directory Structure

```
polylabs/
├── docs/
│   ├── ARCHITECTURE.md              this file
│   └── GRAPH_REFACTOR_SUMMARY.md    graph/DAG refactor history
├── circuits/fl/
│   ├── polylabs_gateway.fl           central gateway + routing
│   ├── polylabs_billing.fl           blinded token subscription graph
│   ├── polylabs_admin.fl             enterprise admin circuits
│   ├── polylabs_tenant.fl            multi-tenant isolation
│   ├── polylabs_sdk_codegen.fl       SDK generation orchestration
│   ├── polylabs_marketplace.fl       component publishing
│   ├── polylabs_metering.fl          usage metering (per-product)
│   ├── polylabs_bridge.fl            opt-in lex bridge management
│   ├── polylabs_onboarding.fl        product provisioning
│   ├── polylabs_compliance.fl        cross-product compliance
│   ├── polylabs_notification.fl      notification routing
│   └── polylabs_analytics.fl         org-level analytics (bridged)
├── sdks/                              FLIR codegen output (generated, not hand-written)
│   ├── polysdk-auth/
│   │   ├── estream-component.toml
│   │   └── dist/                      ← generated
│   ├── polysdk-messenger/
│   ├── polysdk-files/
│   └── polysdk-suite/
├── console/
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── ConsumerDashboard.tsx
│   │   │   └── EnterpriseAdmin.tsx
│   │   └── widgets/                   ← thin glue over PolyKit + codegen'd data
│   │       ├── SubscriptionWidget.tsx
│   │       ├── UsageSummaryWidget.tsx
│   │       ├── ProductLauncherWidget.tsx
│   │       ├── OrgHierarchyWidget.tsx
│   │       ├── RbacManagerWidget.tsx
│   │       └── ComplianceWidget.tsx
│   └── public/
├── marketplace/
│   ├── polysdk-auth/estream-component.toml
│   ├── polysdk-messenger/estream-component.toml
│   ├── polysdk-files/estream-component.toml
│   └── polysdk-suite/estream-component.toml
├── CLAUDE.md
└── package.json
```

Note: No `Cargo.toml` or `crates/` directory. All Rust artifacts are FLIR codegen output, not hand-written crates.

---

## Roadmap — 4-Phase GTM

### Phase 1: Foundation + P0 Products (Q2 2026)

Core infrastructure and highest-priority products launch together:

- **Poly Labs Central**: All 12 FL circuits operational (gateway, billing, admin, tenant, SDK codegen, marketplace, metering, bridge, onboarding, compliance, notification, analytics)
- **Consumer dashboard** with subscription management and product launcher
- **P0 products launch**: Poly Docs, Poly Slides, Poly Messenger, Poly Mail, Poly Git, Poly Files
- **PolySDK codegen pipeline**: `@polysdk/auth` and `@polysdk/suite` generated and published to Marketplace
- **FLIR codegen**: All deployment artifacts (WASM + Rust) produced from FL circuits

### Phase 2: P1 Products + Enterprise (Q3 2026)

Expand product surface and enterprise capabilities:

- **P1 products launch**: Poly Sheets, Poly Notes, Poly Pass, Poly OAuth, Poly VPN, Poly Authenticator, Poly Studio, Poly Comply, Poly Sign, Poly Photos, Poly Contacts, Poly Mind, Poly Wallet
- **Enterprise lex bridge** with k-of-n witness gating
- **Org hierarchy** (composing `group_hierarchy.fl`)
- **RBAC management** (composing `rbac.fl`)
- **Compliance reporting** (GDPR, FedRAMP, CMMC, PCI, SOX, NIST, HIPAA)
- **Per-product PolySDKs** generated on demand via FLIR

### Phase 3: P2 Products + Scale (Q4 2026)

Complete the product family and scale operations:

- **P2 products launch**: Poly Calendar, Poly Tasks, Poly Forms, Poly Phone, Poly Registry, Poly Monitor, Poly Archive, Poly Audit, Poly Vault, Poly Stream, Poly Camera, Poly Find, Poly Health
- **AI-powered churn prediction** (`ai_feed churn_prediction`)
- **Cross-product analytics** (enterprise opt-in only, org-level aggregates)
- **Partner program** (third-party apps embedding PolySDKs)

### Phase 4: Hardware Enhancement + P3 (Q1-Q2 2027)

Companion SoC integration and remaining products:

- **P3 products launch**: Poly Browse, Poly Maps
- **T0 Companion SoC enhancement** for hardware-enhanced products (Poly Wallet, Poly Camera, Poly Find, Poly Health, Poly Files)
- **FPGA acceleration** for scatter-cas operations (Poly Files)
- **Revenue share model** for Marketplace
- **Full 37-product suite** operational with hardware acceleration where applicable
