# PolyQ Labs Suite — Graph/DAG Refactor Summary

**Date**: February 2026 (updated March 2026)
**Platform**: eStream v0.22.0
**QKit**: v0.5.0

---

## Umbrella Principle: Zero-Linkage Privacy Architecture

Every product is cryptographically isolated from every other. A subpoena for one product cannot reveal anything about another — not even that the user has other products.

- Each product gets its own HKDF context producing unlinkable `user_id` values
- Each product has its own StreamSight lex namespace
- Each product has its own metering graph instance
- Billing uses blinded payment tokens — the backend cannot correlate which SPARK identity uses which products

**Individual opt-in**: Any user can choose to link their products for a personal unified dashboard. The bridge is client-side only (WASM + ESLite, derived from `HKDF("q-bridge-v1")`) — no server learns about the linkage. Revocable, content-free, zero subpoena surface.

**Enterprise opt-in**: Enterprise customers can choose to enable org-level cross-product visibility via a lex bridge gated by k-of-n admin witness attestation — revocable, limited to org-level aggregates, explicitly denies user-level identifiers.

---

## What Was Delivered

| Repo | What | Status |
|------|------|--------|
| **polylabs-dev/CLAUDE.md** | Updated top-level with zero-linkage architecture, full repo map | Committed |
| **polyqlabs/** (NEW) | Central monorepo: 12 FL circuits (gateway, billing, admin, tenant, SDK codegen, marketplace, metering, bridge, onboarding, compliance, notification, analytics) + FLIR-codegen'd PolySDKs + console | Pushed to GitHub |
| **qkit/** | Architecture v0.3.0: composes eStream `rbac.fl`/`group_hierarchy.fl`/`issue_tracking.fl`, adds `user_graph` + `metering_graph` (per-app isolated), `subscription_lifecycle` state machine, `lex_bridge` spec | Feature branch pushed |
| **polydata/** | Architecture v3.0: scatter-cas replaces custom CAS, poly-git wraps es-git, `file_registry` graph + `version_history` DAG + `share_network` graph, FastLang .fl replaces legacy YAML | Feature branch pushed |
| **qmessenger/** (NEW) | Fresh repo with 21 screens + 14 hooks + 7 type files extracted from polyquantum/qmessenger-app. `contact_network` graph + `message_thread` DAG + `relay_mesh` graph | Pushed to GitHub |
| **qmail/** | Architecture with `mailbox_registry` graph + `email_thread` DAG + SMTP bridge | Pushed |
| **qvpn/** | Architecture with `vpn_exit_mesh` graph + `tunnel_route` DAG + traffic mimicry | Pushed |
| **qpass/** | Architecture with `vault_registry` graph + `credential_lifecycle` state machine | Pushed |
| **qoauth/** | Architecture with `identity_federation` graph + `token_chain` DAG (ML-DSA-87 PQ-JWT) | Pushed |
| **qmind/** | Architecture with `knowledge_corpus` graph + `legacy_governance` DAG (k-of-n guardian) | Pushed |

---

## eStream Upstream Compositions

Rather than reinventing core platform graphs, PolyQ Labs composes these eStream production `.fl` files:

| eStream Graph | File | PolyQ Labs Usage |
|---------------|------|-----------------|
| `graph rbac` | `estream/circuits/graphs/rbac.fl` | QKit wraps `check_permission()` and `resolve_permissions()` into profiles. Per-product RBAC instances (zero-linkage). Enterprise lex bridge can optionally connect them. |
| `graph group_hierarchy` | `estream/circuits/graphs/group_hierarchy.fl` | `polyqlabs` monorepo composes for enterprise org structure (`OrgNode`, `GroupNode`, `RepoNode`). |
| `state_machine issue_lifecycle` | `estream/circuits/graphs/issue_tracking.fl` | Lifecycle pattern (`guard`, `li_anomaly_detection`, `persistence wal`) reused across all Q apps for entity state machines. |

---

## Graph/DAG Summary Across Suite

| Product | Graphs | DAGs | State Machines |
|---------|--------|------|----------------|
| **QKit** | `user_graph`, `metering_graph` | — | `subscription_lifecycle` |
| **PolyLabs** | `subscription_graph` | — | — |
| **PolyData** | `file_registry`, `share_network` | `version_history` | `share_lifecycle` |
| **PolyMessenger** | `contact_network`, `relay_mesh` | `message_thread` | `message_lifecycle` |
| **PolyMail** | `mailbox_registry` | `email_thread` | `email_lifecycle` |
| **PolyVPN** | `vpn_exit_mesh` | `tunnel_route` | — |
| **PolyPass** | `vault_registry` | — | `credential_lifecycle` |
| **PolyOAuth** | `identity_federation` | `token_chain` | `session_lifecycle` |
| **PolyMind** | `knowledge_corpus` | `legacy_governance` | `ingestion_lifecycle`, `legacy_lifecycle` |

**Totals**: 14 graphs, 7 DAGs, 9 state machines across 9 initial products.

> **March 2026 update**: The product family has expanded from 9 to **37 products across 7 verticals** (Productivity, Communication, Security & Identity, Developer Tools, Enterprise & Compliance, Consumer & Media, Hardware-Enhanced). The graph/DAG patterns established here are reused by all new products via QKit composition. Estimated ~355 product-level FL circuits total. Architecture is now fully FL-native — all hand-written Rust crates (`polyqlabs-gateway`, `polyqlabs-billing`, `polyqlabs-admin`) have been superseded by FLIR codegen from FL circuits. PolySDKs are generated from FL circuit signatures via `service_dispatch` + `tool_registry`. See `ARCHITECTURE.md` for the current system design.

---

## Zero-Linkage Per-Product Isolation

| Product | HKDF Context | Lex Namespace | user_id |
|---------|-------------|---------------|---------|
| Poly Data | `q-data-v1` | `esn/.../polyqlabs/data` | SHA3-256(data_signing_pk)[0..16] |
| Q Messenger | `q-messenger-v1` | `esn/.../polyqlabs/messenger` | SHA3-256(msg_signing_pk)[0..16] |
| Q Mail | `q-mail-v1` | `esn/.../polyqlabs/mail` | SHA3-256(mail_signing_pk)[0..16] |
| Q VPN | `q-vpn-v1` | `esn/.../polyqlabs/vpn` | SHA3-256(vpn_signing_pk)[0..16] |
| Q Pass | `q-pass-v1` | `esn/.../polyqlabs/pass` | SHA3-256(pass_signing_pk)[0..16] |
| Q OAuth | `q-oauth-v1` | `esn/.../polyqlabs/oauth` | SHA3-256(oauth_signing_pk)[0..16] |
| Q Mind | `q-mind-v1` | `esn/.../polyqlabs/mind` | SHA3-256(mind_signing_pk)[0..16] |
| Billing | `billing-v1` | `esn/.../polyqlabs/billing` | blinded_token (no SPARK link) |

The same human has 8 different, unlinkable identities. No circuit, stream, query, or backend system can correlate them.

---

## PolySDKs — Marketplace Components

Embeddable SDKs for third-party apps, published to eStream Marketplace:

| SDK | Package | Description |
|-----|---------|-------------|
| `@polysdk/auth` | `polyqlabs/sdks/polysdk-auth/` | SPARK biometric authentication component |
| `@polysdk/messenger` | `polyqlabs/sdks/polysdk-messenger/` | Embeddable PQ-encrypted messaging |
| `@polysdk/data` | `polyqlabs/sdks/polysdk-data/` | Embeddable scatter-encrypted file storage |
| `@polysdk/suite` | `polyqlabs/sdks/polysdk-suite/` | All-in-one bundle with `<PolySuiteProvider>` |

Each packaged with `estream-component.toml` manifest, WASM bundle, and thin React bindings.

---

## Consumer vs Enterprise

All products share the same codebase. Tier gating is handled by:

| Capability | Consumer | Enterprise |
|---|---|---|
| Auth | SPARK self-service | SPARK + Q OAuth SSO |
| Admin | Per-user settings | Org hierarchy (via `group_hierarchy.fl`), RBAC (via `rbac.fl`), compliance |
| Storage | Per-user scatter | Org-wide scatter with classification policy |
| Billing | Self-service tiers (blinded tokens) | Custom contracts, seat-based |
| Console | Product dashboards | Unified suite console + audit |
| SDKs | Individual product SDKs | Suite SDK with admin hooks |
| Cross-product | Zero-linkage default, personal bridge opt-in (client-side only) | Zero-linkage default, personal + org-level bridge opt-in |
