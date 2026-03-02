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

## Developer Language Story (v0.9.1)

eStream supports **7 languages** at full parity: Rust (native), Python (PyO3), TypeScript (WASM), Go (CGo), C++ (FFI), Swift (C bridging), and FastLang (native).

### External Messaging

- Lead with **"7 supported languages"** — developers choose the language they already know
- Position FastLang as **"the shortest path to silicon"** — the easiest way to design for eStream hardware
- **ESCIR (eStream Circuit Intermediate Representation) is strictly internal** — never mention it in external-facing materials, docs, pitches, or marketing. It is an implementation detail of the compiler
- Swift (not Solidity) is the 7th language

### Internal Development

- **FastLang first**: all new circuits and features are authored in FastLang (.fl) first
- **Six-language parity**: every FastLang feature must have equivalent API surface in Rust, Python, TypeScript, Go, C++, and Swift. Do not ship a FastLang-only feature
- Implementation types: FastLang (.fl), Hybrid (FastLang + Rust/RTL), Pure Rust, Pure RTL, Platform (tooling)
- ESCIR operations power the compiler pipeline but are invisible to users

## Cross-Repo Coordination

This repo is part of the [polylabs-dev](https://github.com/polylabs-dev) organization, coordinated through the **AI Toolkit hub** at `toddrooke/ai-toolkit/`.

For cross-repo context, strategic priorities, and the master work queue:
- `toddrooke/ai-toolkit/CLAUDE-CONTEXT.md` — org map and priorities
- `toddrooke/ai-toolkit/scratch/BACKLOG.md` — master backlog
- `toddrooke/ai-toolkit/repos/polylabs-dev.md` — this org's status summary
