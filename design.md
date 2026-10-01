Design — Distributed API Gateway with Adaptive Routing
1. Design Principles
- Deterministic behavior for equivalent inputs.
- Failure isolation between backends.
- Bounded retries and historical state.
- Atomic configuration changes.
- Recoverability without gateway restart.
2. Logical Flow
Client
  ↓
HTTP Server
  ↓
Route Matcher
  ↓
Eligible Backend Pool
  ↓
Adaptive Selector
  ↓
Upstream
  ↓
Observation / Health / Circuit State
3. Request Lifecycle
Receive → Route → Select → Attempt → Observe → Respond
                         ↓ failure
                      Retry if allowed
4. Backend State
HEALTHY → OPEN/UNHEALTHY → HALF_OPEN → HEALTHY
                         ↘ failure → OPEN
Disabled configuration is separate from health state.
5. Adaptive Routing
Use bounded recent observations. Relevant signals include configured weight, success/failure rate, timeout behavior, and recent latency. The exact algorithm is an implementation choice; the observable contract is that healthy responsive backends remain eligible, failing/slow backends can be deprioritized, and recovered backends can return to service.
6. Retry Model
Each request has a finite attempt budget and deadline. A failed attempt records its outcome before another eligible backend is selected. The gateway must guarantee:
- attempts never exceed the configured limit;
- elapsed time respects the request deadline;
- unsafe requests are not blindly retried.
7. Configuration
Treat configuration as versioned immutable snapshots:
parse → validate → publish atomically
                 ↘ invalid → retain current
New requests use the new valid snapshot; in-flight requests must not see partial updates.
8. Concurrency
Independent requests should execute concurrently. Shared state must remain consistent during traffic, health changes, reloads, and shutdown.
9. Shutdown
RUNNING → DRAINING → STOPPED
Draining stops new work according to the contract, allows active work to finish within a bound, and stops background activities.
10. Verifier Design
Use controlled local upstream fixtures capable of returning success, errors, delays, and connection failures. Test state transitions rather than isolated calls.
Recommended families:
1. lifecycle
2. route matching
3. backend selection
4. adaptive routing
5. health
6. circuit breaking
7. retries
8. timeouts
9. configuration reload
10. concurrency
11. shutdown
12. determinism
13. performance
11. Mutants
Useful realistic mutants:
- incorrect prefix precedence;
- ignored weights;
- latency adaptation disabled;
- health never updated;
- circuit never opens;
- retry off-by-one;
- unsafe retry;
- timeout ignored;
- invalid configuration replaces valid configuration;
- partial reload;
- immediate shutdown;
- unbounded observations.