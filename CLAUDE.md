# Poly Labs Central

**GitHub**: [polylabs-dev/polylabs](https://github.com/polylabs-dev/polylabs)
**Platform**: eStream v0.8.3
**Depends on**: PolyKit v0.3.0, eStream graph/DAG constructs

## Purpose

Central monorepo for Poly Labs: backend services (gateway, billing, enterprise admin), PolySDKs (embeddable components for third-party apps), and the unified console (consumer dashboard + enterprise admin).

## Zero-Linkage Privacy

This repo enforces the zero-linkage privacy architecture. The billing system uses **blinded payment tokens** derived from a separate HKDF context (`billing-v1`). No backend system in this repo can correlate user identities across products. Enterprise customers can opt-in to cross-product org-level visibility via a lex bridge gated by k-of-n admin witness attestation.

## Structure

- `crates/` — Rust backend services (gateway, billing, admin)
- `sdks/` — PolySDKs (`@polysdk/auth`, `@polysdk/messenger`, `@polysdk/data`, `@polysdk/suite`)
- `console/` — React management UI (consumer + enterprise)
- `circuits/fl/` — FastLang circuit definitions
- `marketplace/` — eStream Marketplace component manifests
- `docs/` — Architecture and design documents

## Commit Convention

Commit to the GitHub issue or epic the work was done under. Do not accumulate large amounts of uncommitted work.
