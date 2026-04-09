# Epic: PolyLabs Backend — CE + App Graph Integration

> **Repo**: `polyqlabs`
> **Spec**: `specs/POLYLABS_CE_APP_GRAPH_SPEC.md`
> **Priority**: P0
> **Status**: In Progress

---

## Summary

Integrate the Cognitive Engine and App Graph into the PolyLabs central backend. This registers all 12 backend modules (gateway 3, billing 4, admin 3, graphs 2) into the Stratum module graph, wires 18 dependency edges plus 4 cross-graph bridges, and configures 3 CE meaning domains with noise filtering and SME panels.

---

## Tasks

### Phase 1: App Graph Registration

- [ ] **P1.1** — Implement `polyqlabs_app_graph.fl` with 12 module definitions and `make_polyqlabs_module` helper
- [ ] **P1.2** — Wire 18 `EDGE_REQUIRES` dependency edges in `polyqlabs_app_graph_register`
- [ ] **P1.3** — Implement `polyqlabs_register_bridge_edges` (4 bridges: QKit metering, QKit RBAC, SPARK, scatter-cas)
- [ ] **P1.4** — Implement `polyqlabs_register_governance_edges` (12 `EDGE_GOVERNANCE_OBSERVE`)
- [ ] **P1.5** — Add `find_polyqlabs_module_id` helper and `polyqlabs_app_graph_metrics` store
- [ ] **P1.6** — Add safety/liveness properties and 3 golden tests for graph registration

### Phase 2: CE Meaning Domains

- [ ] **P2.1** — Define `GatewayHealthDomain`, `BillingPatternsDomain`, `TenantLifecycleDomain` data types with cortex blocks
- [ ] **P2.2** — Implement `register_gateway_health_domain` circuit (threshold 70, impact 80)
- [ ] **P2.3** — Implement `register_billing_patterns_domain` circuit (threshold 80, impact 90)
- [ ] **P2.4** — Implement `register_tenant_lifecycle_domain` circuit (threshold 75, impact 70)
- [ ] **P2.5** — Add observation streams (`gateway_health_obs`, `billing_pattern_obs`, `tenant_lifecycle_obs`)

### Phase 3: Noise Filter & SME Panels

- [ ] **P3.1** — Define `PlatformNoiseFilterConfig` data type
- [ ] **P3.2** — Implement `configure_platform_noise_filter` (suppress health checks, probes, synthetic traffic)
- [ ] **P3.3** — Define `GatewayOptimizationPanel` and `BillingEvolutionPanel` data types
- [ ] **P3.4** — Implement `configure_gateway_optimization_panel` (4 specializations, 75% calibration floor)
- [ ] **P3.5** — Implement `configure_billing_evolution_panel` (4 specializations, 80% calibration floor)
- [ ] **P3.6** — Add `full_ce_pipeline_setup` golden test

### Phase 4: Integration & Validation

- [ ] **P4.1** — Verify all 12 modules resolve via `find_polyqlabs_module_id` after registration
- [ ] **P4.2** — Verify bridge edges connect to live QKit and eStream module graphs
- [ ] **P4.3** — Verify governance observer can read all 12 modules
- [ ] **P4.4** — Verify CE domains produce observations that flow through noise filter to cortex
- [ ] **P4.5** — Verify SME panels accept and adjudicate crystallization candidates
- [ ] **P4.6** — Run full FLIR codegen (WASM + Rust targets) on both circuit files

---

## Acceptance Criteria

1. `polyqlabs_app_graph_register` produces a `CsrStorage` with exactly 12 nodes and 18 edges
2. `polyqlabs_register_bridge_edges` adds 4 bridge nodes with `EDGE_BRIDGE_TO` edges
3. `polyqlabs_register_governance_edges` adds 12 `EDGE_GOVERNANCE_OBSERVE` edges
4. All 3 meaning domains register with correct crystallization thresholds
5. Noise filter suppresses kube-probe, ELB-HealthChecker, GoogleHC user agents
6. Both SME panels require min 3 panelists with calibration floors >= 70%
7. All golden tests pass under `fl test --golden`
8. Both files compile to WASM and Rust via `fl build --target wasm,rust`

---

## Files

| File | Description |
|------|-------------|
| `circuits/fl/polyqlabs_app_graph.fl` | 12-module app graph, edges, bridges, governance, tests |
| `circuits/fl/polyqlabs_meaning.fl` | 3 CE domains, noise filter, 2 SME panels, tests |
| `specs/POLYLABS_CE_APP_GRAPH_SPEC.md` | Integration spec |
