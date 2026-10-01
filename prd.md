PRD — Distributed API Gateway with Adaptive Routing
1. Product Overview
Project: Distributed API Gateway with Adaptive Routing
Benchmark: RL Gym / RLVR / AI Evaluation
Build a local, testable HTTP gateway that routes requests to configured upstreams, tracks backend health and latency, and adapts routing based on observed behavior.
2. Problem
A static reverse proxy cannot reliably handle backend failures, latency changes, configuration changes, retries, and concurrent traffic. This benchmark evaluates whether an autonomous coding agent can implement these stateful workflows correctly.
3. Goals
- HTTP request routing
- Multiple backend pools
- Weighted and adaptive routing
- Health tracking and recovery
- Bounded retries
- Upstream timeouts
- Circuit breaking
- Atomic configuration reload
- Graceful shutdown
- Concurrent request handling
- Deterministic observable behavior
4. Non-Goals
- Kubernetes integration
- External cloud discovery
- TLS certificate management
- HTTP/3
- Full WAF
- Machine-learning routing
- Distributed consensus
5. Core Requirements
Routing
Support exact paths, path prefixes, multiple routes, method constraints, deterministic precedence, and a defined no-route response.
Backend Pools
Each backend has an ID, address, weight, enabled/disabled state, and runtime health state. Disabled or protected backends receive no ordinary traffic.
Adaptive Routing
Selection considers configured weight, recent failures, recent latency, and eligibility. Adaptation is bounded; one anomalous request cannot permanently dominate routing.
Health
Repeated failures can make a backend unhealthy. Unhealthy backends are protected from ordinary traffic. Recovery probes can return a backend to service.
Retries
Retries are finite and policy-controlled. Unsafe requests must not be blindly retried after possible side effects. Retry storms are prohibited.
Timeouts
Upstream timeouts terminate an attempt, update state, and may trigger a permitted retry.
Circuit Breaking
Repeated failures move a backend into a protected state. A later bounded recovery attempt can transition it back to service.
Configuration Reload
Valid configuration becomes active atomically. Invalid configuration is rejected and the last valid configuration remains active. In-flight requests must not observe partial configuration.
Graceful Shutdown
New work stops according to the shutdown contract, active work gets a bounded opportunity to finish, and background work terminates.
6. Stateful Workflows
- Healthy backend → failure → unhealthy → recovery → healthy.
- Normal routing → sustained latency difference → adaptive routing → recovery.
- Configuration A → valid B.
- Configuration A → invalid B → A remains active.
- Upstream failure → bounded retry → success/final error.
- Running → draining → stopped.
7. Reliability
The gateway must not crash on malformed configuration or upstream failures, must not retry indefinitely, and must bound runtime observation state.
8. Evaluation
The private verifier should launch local upstream servers, control delays/failures, reload configuration, issue concurrent requests, and observe externally visible results. It must not inspect internal implementation details.
Reference solution: 100% of score-bearing checks. Mutants must be caught by black-box tests.