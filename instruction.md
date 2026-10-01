Model Instruction — Distributed API Gateway with Adaptive Routing
Your Task
Implement the software described below in the repository.
You are responsible for producing a complete working solution, not merely a design or prototype.
The evaluator will interact with your implementation through its public lifecycle and runtime interfaces and will judge observable behavior.
Do not modify or depend on private evaluation assets.
Problem
Build a local HTTP API gateway with adaptive routing.
The gateway receives client HTTP requests and routes them to configured upstream backend services. It must react to backend failures and latency changes, support bounded retries and timeouts, protect repeatedly failing backends, support live configuration reload, and shut down gracefully.
The gateway must run locally without external cloud services.
What Must Work
1. Lifecycle
The repository must support the benchmark lifecycle:
- build the application;
- start the gateway;
- reset generated/runtime state.
The application must start successfully in the supplied evaluation environment.
2. HTTP Routing
Support multiple configured routes.
Routes can match:
- HTTP method;
- exact path;
- path prefix.
When multiple routes match, use deterministic precedence:
1. exact path;
2. longest path prefix;
3. method-specific match;
4. stable configuration order.
A request with no matching route must receive the defined gateway no-route response.
3. Backend Pools
Routes may have multiple upstream backends.
Each backend has an identifier, address, weight, and enabled state.
Disabled backends must not receive normal traffic.
The gateway must track runtime health separately for each backend.
4. Backend Selection
Only eligible backends may be selected.
Eligibility excludes backends that are disabled or currently protected by the circuit-breaker/health policy.
Configured weights must influence routing when backends have otherwise equivalent performance.
Selection must be deterministic for equivalent state.
5. Adaptive Routing
The gateway must learn from recent runtime behavior.
Routing must account for:
- configured weight;
- recent latency;
- recent failures;
- recent timeouts;
- health/circuit state.
A consistently slower backend must be treated differently from an equally fast backend.
A repeatedly failing backend must eventually receive less traffic or become protected.
When a backend recovers, it must be capable of returning to normal routing.
A single abnormal request must not permanently change a backend's routing behavior.
Historical observations must be bounded.
6. Health and Circuit Breaking
Repeated failures or timeouts must be able to move a backend into a protected state.
Protected backends must not receive ordinary traffic.
After a configured recovery interval, the gateway must permit a bounded recovery attempt.
A successful recovery restores normal eligibility.
A failed recovery keeps the backend protected.
Only the affected backend should be penalized; another healthy backend must remain usable.
7. Upstream Timeout
Every upstream attempt must have a finite timeout.
When an attempt times out:
- terminate the attempt;
- record the timeout;
- update backend state;
- retry only when permitted;
- otherwise return a deterministic gateway error.
8. Retries
Retries must be finite.
Never exceed the configured maximum number of attempts.
Retry only when the request and failure satisfy the configured retry policy.
Do not blindly retry potentially side-effecting requests when the gateway cannot determine whether the upstream processed the request.
When possible, a retry should use another eligible backend.
9. Configuration Reload
The running gateway must support configuration reload.
A valid configuration must become active atomically for subsequent requests.
An invalid configuration must be rejected and must not replace the last valid configuration.
Requests must never observe a partially applied configuration.
A reload must not corrupt requests that are already in flight.
10. Graceful Shutdown
Support graceful shutdown.
During shutdown:
- new work must stop according to the documented contract;
- active work receives a bounded opportunity to finish;
- background work terminates;
- the gateway exits within a bounded time.
11. Concurrency
Handle multiple concurrent requests safely.
The following may happen concurrently:
- requests;
- backend failures;
- backend recovery;
- configuration reload;
- graceful shutdown.
These operations must not corrupt shared state, deadlock, or cause unbounded retries.
12. Failure Handling
The process must remain alive after upstream:
- connection failures;
- timeouts;
- unexpected connection closes;
- error responses.
Malformed configuration must be rejected without destroying the active valid configuration.
Required Stateful Behaviors
Your implementation must support these transitions:
healthy backend
    ↓
repeated failures
    ↓
protected backend
    ↓
recovery attempt
    ↓
healthy backend
and:
valid configuration A
    ↓
valid configuration B
    ↓
new requests use B
and:
valid configuration A
    ↓
invalid configuration B
    ↓
A remains active
and:
upstream failure
    ↓
bounded retry
    ↓
success or deterministic final failure
Edge Cases
Correctly handle:
- no eligible backends;
- all backends failing;
- one backend recovering while traffic continues;
- invalid reload after a valid configuration;
- reload during active requests;
- timeout during shutdown;
- concurrent backend failures;
- exact-path/prefix conflicts;
- method mismatch;
- invalid weights;
- duplicate backend IDs;
- invalid backend addresses;
- empty route configuration;
- retryable and non-retryable upstream errors.
Implementation Freedom
You decide how to implement the system.
The task does not require a particular:
- language;
- framework;
- module structure;
- routing library;
- concurrency mechanism;
- health-check implementation;
- adaptive-routing formula;
- persistence technology.
The evaluator cares about externally observable behavior.
Important Constraint
Do not implement only the happy path.
The evaluation focuses heavily on state transitions, failure recovery, concurrent behavior, adaptive routing, bounded retries, and configuration safety.
Before considering the implementation complete, exercise the gateway against changing backend health, changing latency, configuration reloads, timeouts, retries, and graceful shutdown.