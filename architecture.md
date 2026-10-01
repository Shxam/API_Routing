Architecture — Distributed API Gateway with Adaptive Routing
1. Reference Architecture
Client
  ↓
HTTP Server
  ↓
Route Matcher
  ↓
Backend Eligibility
  ↓
Adaptive Selector
  ↓
Upstream Executor
  ↓
Observation Store
  ├── Health Manager
  └── Circuit Breaker

Configuration Manager ──→ immutable configuration snapshots
Shutdown Manager ────────→ lifecycle coordination
2. Components
HTTP Server
Accepts requests, manages request lifecycle, invokes routing, returns responses, and participates in shutdown.
Configuration Manager
Loads and validates configuration and publishes valid versions atomically. The invariant is that the active configuration is always valid.
Route Matcher
Determines the effective route using exact/prefix/method rules and documented precedence.
Backend Pool
Tracks configured backends and determines which are enabled and eligible.
Adaptive Selector
Selects an eligible backend using configured weight plus bounded runtime observations. It should not perform network I/O.
Upstream Executor
Executes upstream requests, enforces timeouts, and classifies responses/network failures.
Retry Controller
Maintains finite attempt budgets, determines retry eligibility, and selects subsequent candidates.
Health Manager
Records successes, failures, timeouts, and recovery outcomes and transitions backend health.
Circuit Breaker
Protects repeatedly failing backends and controls bounded recovery probes.
Observation Store
Maintains bounded recent latency and outcome statistics.
Shutdown Manager
Coordinates RUNNING → DRAINING → STOPPED and ensures background work terminates.
3. Configuration Model
A conceptual configuration may contain:
routes:
  - id: users
    match:
      path_prefix: /users
      methods: [GET]
    backends:
      - id: users-a
        address: http://127.0.0.1:9001
        weight: 100
      - id: users-b
        address: http://127.0.0.1:9002
        weight: 100

retry:
  max_attempts: 2
timeout:
  upstream_ms: 500
health:
  failure_threshold: 3
  recovery_ms: 1000
adaptive:
  window_size: 50
The final schema must remain consistent across task, reference, and verifier.
4. Configuration Reload
disk/config
   ↓
parse
   ↓
validate ── invalid ──→ keep current
   ↓
new snapshot
   ↓
atomic publication
Requests should use a complete configuration snapshot rather than observing partial changes.
5. Adaptive Routing
A reference implementation may use a bounded effective score conceptually similar to:
configured_weight × health_factor × latency_factor
The exact formula is not a model-facing requirement. Runtime observations must be bounded so a single anomalous request cannot permanently dominate.
6. Retry Architecture
For each request:
1. select eligible backend;
2. execute within deadline;
3. record outcome;
4. return on success;
5. classify failure;
6. retry only if allowed;
7. stop at attempt or deadline limit.
7. Failure Isolation
Backend A failure may change A's health, circuit, and adaptive state, but must not corrupt B/C, route configuration, or gateway lifecycle.
8. Administrative Observability
If the finalized benchmark includes an administrative endpoint, it should expose only stable documented state such as backend health, configuration version, and routing statistics. It must not expose implementation internals.
9. Black-Box Test Topology
Verifier
 ├── Gateway
 ├── Upstream A (success/failure/delay)
 └── Upstream B (success/failure/delay)
Tests control upstream behavior and observe client-visible gateway behavior.
10. Architecture Boundary
This architecture is for the reference solution. The model-facing task must not require specific modules, data structures, frameworks, adaptive formulas, or concurrency primitives. The evaluator tests behavior, not architectural similarity.