# Poly Labs Central Architecture

**Version**: 1.0.0
**Date**: February 2026
**Platform**: eStream v0.8.3
**Upstream**: PolyKit v0.3.0, eStream graph/DAG constructs

---

## Overview

The `polylabs` repo is the **central monorepo** for Poly Labs. It contains:

- **Backend services** — gateway, billing, enterprise admin
- **PolySDKs** — embeddable SDKs for third-party apps to integrate Poly products
- **Console** — unified management UI (consumer dashboard + enterprise admin)
- **Marketplace components** — PolySDK packages published to eStream Marketplace

All products (Poly Data, Poly Messenger, Poly Mail, etc.) are independent repos. This repo provides the **cross-cutting infrastructure** that connects them -- without violating zero-linkage privacy.

---

## Zero-Linkage Privacy Architecture

### The Problem

A privacy-focused product suite creates a tension: users want a unified experience, but cross-product linkage creates a subpoena surface. If the backend can prove that user X uses both Poly Messenger and Poly Data, then a legal request for Messenger data implicitly reveals the existence of their Data files.

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
              │data-v1  │ │msg-v1  │ │  │mail-v1     │
              │user_id_A│ │user_id_B│ │  │user_id_C   │
              └─────────┘ └────────┘ │  └────────────┘
                                     │
                            ┌────────┴────────┐
                            │billing-v1       │
                            │blinded_token    │
                            └─────────────────┘
```

- Each product derives an independent `user_id` from a product-specific HKDF context
- The billing system receives a blinded payment token, not a SPARK identity
- No backend system can correlate user_id_A (Poly Data) with user_id_B (Poly Messenger)
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

    source esn/global/org/polylabs/data/{user_id_A}
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

How it works:
1. User derives a **bridge key** from SPARK master seed via `HKDF("poly-bridge-v1")` — a new, dedicated derivation context
2. The bridge key signs **linkage proofs**: `sign(bridge_key, user_id_A || user_id_B || user_id_C)` — proving the same human controls all three product identities
3. Linkage proofs are stored **client-side only** (ESLite, encrypted with the bridge key). The backend never receives them.
4. The personal dashboard reads aggregate stats from each product's local ESLite cache and presents them in a unified view — all in WASM, no server round-trip
5. If the user revokes the bridge, the linkage proofs are deleted from ESLite. Products return to fully isolated state.

Key properties:
- **Client-side only**: The bridge exists in WASM + ESLite. No server or lattice node learns that these identities belong to the same human.
- **Cryptographically bound**: Only the SPARK master seed holder can create or revoke the bridge.
- **Content-free**: The bridge passes usage aggregates (storage used, message count, etc.) but never content, contacts, or encryption keys.
- **Revocable**: Deleting the bridge key material from ESLite severs the link permanently.
- **No subpoena surface**: Since the linkage exists only on the user's device, there is nothing on any server to compel.

### Enterprise Opt-In Bridge

Enterprise admins can choose to enable cross-product visibility at the org level:

```fastlang
lex_bridge polylabs_enterprise {
    scope organization
    owner org_admin

    source esn/global/org/polylabs/data
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

The enterprise bridge:
- Requires k-of-n admin witness attestation (3-of-5 admin keys must sign)
- Only passes org-level aggregates (seat count, storage used, compliance status)
- Explicitly denies user-level identifiers and content
- Is revocable at any time
- Audited via tamper-proof series

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Poly Labs Central (polylabs/)                                          │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Console (React + @polykit/react)                                 │  │
│  │  ├── Consumer Dashboard    subscription, usage, product links     │  │
│  │  ├── Enterprise Admin      org mgmt, RBAC, compliance, audit     │  │
│  │  └── Widget Registry       per-product widgets + cross-product   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  PolySDKs (WASM + thin React bindings)                            │  │
│  │  ├── @polysdk/auth         SPARK biometric auth component         │  │
│  │  ├── @polysdk/messenger    embeddable messaging                   │  │
│  │  ├── @polysdk/data         embeddable file browser/storage        │  │
│  │  └── @polysdk/suite        all-in-one bundle                      │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Backend Crates (Rust, lattice-hosted)                            │  │
│  │  ├── polylabs-gateway      eStream wire protocol gateway          │  │
│  │  ├── polylabs-billing      blinded token billing circuits         │  │
│  │  └── polylabs-admin        enterprise admin circuits              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  FastLang Circuits                                                │  │
│  │  ├── polylabs_billing.fl            subscription graph            │  │
│  │  ├── polylabs_admin.fl              enterprise admin              │  │
│  │  └── polylabs_gateway.fl            central gateway               │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  Composes: PolyKit v0.3.0 + eStream group_hierarchy.fl + rbac.fl      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Graph/DAG Constructs

### Subscription Graph (`polylabs_billing.fl`)

Manages subscriptions using blinded tokens. The graph does NOT store SPARK identities or product-specific user_ids. It operates on anonymous payment tokens.

```fastlang
type BlindedAccountNode = struct {
    token_hash: bytes(32),
    tier: u8,
    created_at: u64,
    billing_cycle_start: u64,
}

type ProductEntitlementNode = struct {
    entitlement_id: bytes(16),
    product_code: u8,
    tier: u8,
    valid_from: u64,
    valid_until: u64,
}

type SubscribesEdge = struct {
    subscribed_at: u64,
    payment_method_hash: bytes(32),
}

graph subscription_graph {
    node BlindedAccountNode
    node ProductEntitlementNode
    edge SubscribesEdge

    overlay tier: u8 curate delta_curate
    overlay mrr_cents: u64 bitmask delta_curate
    overlay churn_risk: u32 bitmask delta_curate
    overlay last_payment_ns: u64 curate

    storage csr {
        hot @bram,
        warm @ddr,
        cold @nvme,
    }

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

Key constraint: `BlindedAccountNode.token_hash` is derived from `HKDF("billing-v1")` -- a separate derivation context that cannot be correlated to any product-specific identity. The billing system knows "account X pays for tier Y" but cannot determine which SPARK user or which product user_ids belong to account X.

### Enterprise Org Graph (composes `group_hierarchy.fl`)

For enterprise customers who opt in, the `polylabs_admin.fl` circuit extends eStream's `group_hierarchy.fl`:

```fastlang
type PolyOrgExtension = struct {
    org_id: bytes(32),
    licensed_products: u16,
    seat_limit: u32,
    contract_tier: u8,
    contract_expires: u64,
}

circuit assign_product_license(hierarchy: bytes(32), org_id: bytes(32), product_code: u8) -> bool
    lex esn/global/org/polylabs/admin
    precision C
    rbac [org_admin]
    povc true
    observe metrics: [org_id, product_code]
{
    true
}

circuit enforce_seat_limit(hierarchy: bytes(32), org_id: bytes(32)) -> bool
    lex esn/global/org/polylabs/admin
    precision B
    observe metrics: [org_id, seat_count, seat_limit]
{
    true
}

circuit org_compliance_report(hierarchy: bytes(32), org_id: bytes(32)) -> bytes
    lex esn/global/org/polylabs/admin/compliance
    precision B
    rbac [compliance_admin]
    observe metrics: [org_id, report_type]
{
    bytes(0)
}
```

---

## PolySDKs

Embeddable SDKs that let third-party apps integrate Poly products. Each SDK is a self-contained WASM bundle + thin React bindings, published as an eStream Marketplace component.

### SDK Architecture

```
Third-Party App
  └── <PolySuiteProvider> or <PolyAuthProvider>
        └── polykit.wasm (loaded once)
              ├── @polysdk/auth     — SPARK biometric, key derivation
              ├── @polysdk/messenger — blind relay messaging
              ├── @polysdk/data     — scatter file storage
              └── Product-specific WASM circuits
```

### `@polysdk/auth` — SPARK Authentication

Drop-in SPARK biometric authentication for any app.

```tsx
import { SparkAuth, useSparkIdentity } from '@polysdk/auth';

function LoginScreen() {
  const { authenticate, identity, isAuthenticated } = useSparkIdentity({
    hkdfContext: 'my-app-v1',
  });

  return <SparkAuth onAuthenticated={authenticate} />;
}
```

### `@polysdk/messenger` — Embeddable Messaging

Drop a full messaging UI into any app, or use headless hooks.

```tsx
import { PolyMessenger, usePolyMessage } from '@polysdk/messenger';

// Full UI
<PolyMessenger contactId={sparkDid} />

// Headless
const { send, messages, subscribe } = usePolyMessage(contactId);
await send({ text: 'Hello', classification: 'INTERNAL' });
```

### `@polysdk/data` — Embeddable File Storage

Scatter-encrypted file management.

```tsx
import { PolyDataBrowser, usePolyFile } from '@polysdk/data';

// Full UI
<PolyDataBrowser path="/documents" classification="CONFIDENTIAL" />

// Headless
const { upload, download, share } = usePolyFile();
await upload(file, { classification: 'INTERNAL', scatter: '3-of-5' });
```

### `@polysdk/suite` — Full Bundle

All SDKs in one provider, for apps that want the complete Poly experience.

```tsx
import { PolySuiteProvider } from '@polysdk/suite';

<PolySuiteProvider
  products={['messenger', 'data', 'auth']}
  hkdfContext="partner-app-v1"
>
  <App />
</PolySuiteProvider>
```

### Marketplace Packaging

Each SDK is published to eStream Marketplace with an `estream-component.toml`:

```toml
[component]
name = "@polysdk/messenger"
version = "1.0.0"
category = "sdk"
description = "Embeddable PQ-encrypted messaging — blind relay, CRDT sync, media streams"
license = "Commercial"
publisher = "polylabs"
keywords = ["messaging", "pq-crypto", "blind-relay", "sdk", "embeddable"]

[component.targets]
wasm = true
fpga = false

[component.dependencies]
polykit = "^0.3.0"

[component.sdk]
entry_wasm = "dist/polysdk-messenger.wasm"
entry_react = "dist/index.js"
framework = "react"
supports_react_native = true
```

---

## Console

The Poly Labs console is the unified management interface. It serves two audiences:

### Consumer Dashboard

- Product launcher (links to individual product UIs)
- Subscription management (tier, payment method -- all via blinded tokens)
- Usage overview (per-product metering, no cross-product correlation)
- Account settings (SPARK devices, guardians, recovery)

### Enterprise Admin (opt-in bridge required)

- Org management (via eStream `group_hierarchy.fl`)
- RBAC management (via eStream `rbac.fl`)
- Seat management and product licensing
- Compliance reporting (SOC2, GDPR, CCPA audit trails)
- Cross-product usage aggregates (org-level only, never user-level)

### Console Widgets

| Widget | Audience | Data Source |
|--------|----------|-------------|
| `polylabs-subscription` | Consumer | `subscription_graph` |
| `polylabs-usage-summary` | Consumer | Per-product metering (isolated) |
| `polylabs-device-manager` | Consumer | Per-product `user_graph` (isolated) |
| `polylabs-org-hierarchy` | Enterprise | `group_hierarchy.fl` (bridged) |
| `polylabs-rbac-manager` | Enterprise | `rbac.fl` (bridged) |
| `polylabs-seat-manager` | Enterprise | `polylabs_admin.fl` |
| `polylabs-compliance` | Enterprise | Bridged audit streams |

---

## Directory Structure

```
polylabs/
├── docs/
│   └── ARCHITECTURE.md              this file
├── crates/
│   ├── polylabs-gateway/             eStream wire protocol gateway
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── polylabs-billing/             blinded token billing
│   │   ├── Cargo.toml
│   │   └── src/
│   └── polylabs-admin/               enterprise admin
│       ├── Cargo.toml
│       └── src/
├── sdks/
│   ├── polysdk-auth/                 SPARK auth SDK
│   │   ├── package.json
│   │   ├── estream-component.toml
│   │   └── src/
│   ├── polysdk-messenger/            messaging SDK
│   │   ├── package.json
│   │   ├── estream-component.toml
│   │   └── src/
│   ├── polysdk-data/                 file storage SDK
│   │   ├── package.json
│   │   ├── estream-component.toml
│   │   └── src/
│   └── polysdk-suite/                all-in-one bundle
│       ├── package.json
│       ├── estream-component.toml
│       └── src/
├── console/
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── ConsumerDashboard.tsx
│   │   │   └── EnterpriseAdmin.tsx
│   │   └── widgets/
│   │       ├── SubscriptionWidget.tsx
│   │       ├── UsageSummaryWidget.tsx
│   │       ├── OrgHierarchyWidget.tsx
│   │       ├── RbacManagerWidget.tsx
│   │       └── ComplianceWidget.tsx
│   └── public/
├── circuits/fl/
│   ├── polylabs_billing.fl            subscription graph
│   ├── polylabs_admin.fl              enterprise admin circuits
│   └── polylabs_gateway.fl            central gateway circuit
├── marketplace/
│   ├── polysdk-auth/estream-component.toml
│   ├── polysdk-messenger/estream-component.toml
│   ├── polysdk-data/estream-component.toml
│   └── polysdk-suite/estream-component.toml
├── CLAUDE.md
├── Cargo.toml
└── package.json
```

---

## Roadmap

### Phase 1: Foundation (Q2 2026)
- Billing circuits with blinded token model
- Consumer dashboard with subscription management
- `@polysdk/auth` -- SPARK authentication SDK (marketplace component)
- Gateway circuit for product routing

### Phase 2: SDKs (Q3 2026)
- `@polysdk/messenger` -- embeddable messaging
- `@polysdk/data` -- embeddable file storage
- `@polysdk/suite` -- all-in-one bundle
- Marketplace publishing for all SDKs

### Phase 3: Enterprise (Q4 2026)
- Enterprise lex bridge with k-of-n witness gating
- Org hierarchy (composing `group_hierarchy.fl`)
- RBAC management (composing `rbac.fl`)
- Compliance reporting and audit trails
- Seat management and product licensing

### Phase 4: Scale (Q1 2027)
- AI-powered churn prediction (`ai_feed churn_prediction`)
- Cross-product analytics (enterprise opt-in only, org-level aggregates)
- Partner program (third-party apps embedding PolySDKs)
- Revenue share model for marketplace
