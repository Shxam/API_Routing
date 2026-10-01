Task Specification — Distributed API Gateway with Adaptive Routing
1. Objective
Implement a Distributed API Gateway with Adaptive Routing.
The gateway is a local HTTP service that receives client requests, matches them against configured routes, selects an eligible upstream backend, forwards the request, and adapts future routing decisions based on backend health and observed latency.
The implementation is evaluated through black-box integration tests. The tests interact with the application through its documented command-line, HTTP, configuration, and lifecycle interfaces.
You may choose your internal architecture, programming language, libraries, concurrency primitives, data structures, and algorithms unless explicitly constrained below.
2. Required Repository Behavior
The repository must provide a runnable implementation with the following lifecycle:
build
start
reset
The repository must provide whatever root commands are required by the benchmark harness to build, run, reset, and test the application.
The gateway must be able to run entirely on localhost. Tests must not require external cloud services.
3. Configuration
The gateway must load a configuration containing:
- listening address;
- routes;
- upstream backends;
- backend weights;
- retry policy;
- timeout policy;
- health policy;
- circuit-breaker policy;
- adaptive-routing policy.
Use the repository's supplied configuration format if one already exists. If the repository does not provide one, use a human-readable configuration format and document it.
A valid configuration must be validated before becoming active.
Configuration invariants
A configuration is invalid when, for example:
- a route has no usable backend;
- a backend address is malformed;
- a weight is invalid;
- a timeout is invalid;
- retry limits are invalid;
- required fields are missing;
- duplicate backend identifiers occur where uniqueness is required.
An invalid configuration must not replace the currently active valid configuration.
4. HTTP Routing
4.1 Routes
The gateway must support multiple routes.
A route may specify:
- HTTP method(s);
- exact path;
- path prefix;
- backend pool.
The route decision must be deterministic.
4.2 Route precedence
When more than one route matches, use the following precedence:
1. exact path match;
2. longest matching path prefix;
3. method-specific match over a route that accepts all methods;
4. stable configuration order as the final tie-breaker.
A request with no matching route must return the documented no-route response.
5. Backend Pools
A backend has:
- unique identifier;
- HTTP address;
- configured weight;
- enabled/disabled state;
- runtime health state;
- circuit state;
- runtime performance observations.
A disabled backend must never be selected for ordinary traffic.
A backend that is protected by the circuit breaker must not receive ordinary traffic.
If all backends are unavailable, the gateway must return a deterministic gateway error.
6. Backend Selection
For each request, the gateway must construct an eligible backend set.
A backend is eligible only when it is:
- configured;
- enabled;
- not protected by an open circuit;
- available according to the current health policy.
The selector must use configured backend weights.
When two or more backends are otherwise equivalent, selection must use a deterministic tie-breaking rule rather than uncontrolled map/object iteration.
7. Adaptive Routing
The gateway must adapt backend selection based on runtime observations.
The adaptive policy must consider at least:
- configured weight;
- recent successful/failed requests;
- recent timeouts;
- recent latency;
- health/circuit state.
Required behavior
If backend A and backend B are both healthy and have equivalent observed performance, configured weights must influence their traffic share.
If backend A becomes consistently slower than backend B, future traffic must adapt according to the configured adaptive policy so that A is not treated as having identical performance indefinitely.
If backend A repeatedly fails, it must receive progressively less traffic and may become ineligible through the health/circuit policy.
If A recovers, the system must allow it to become eligible again.
Bounded adaptation
A single anomalous request must not permanently suppress a backend.
Historical observations must be bounded. The gateway must not maintain unbounded per-backend request history.
8. Health Tracking
Backend health is stateful.
The implementation must track outcomes across requests.
At minimum, distinguish:
- healthy;
- degraded/unhealthy;
- recovery/probe state.
Repeated failures or timeouts must cause the backend to become protected according to the configured failure threshold.
Once protected, it must not receive ordinary requests.
After the configured recovery interval, the gateway must permit a bounded recovery attempt.
Recovery success
A successful recovery attempt must allow the backend to return to normal eligibility.
Recovery failure
A failed recovery attempt must return the backend to the protected state.
One backend's failure must not corrupt the health state of other backends.
9. Circuit Breaker
The circuit breaker must provide behavior equivalent to:
CLOSED
  |
  | failure threshold
  v
OPEN
  |
  | recovery interval
  v
HALF_OPEN
  |       \
 success   failure
  |          |
  v          v
CLOSED      OPEN
CLOSED
Normal requests may be sent according to routing policy.
OPEN
Normal traffic must not be sent to the backend.
HALF_OPEN
Only a bounded recovery/probe request may be allowed.
A successful probe closes the circuit.
A failed probe reopens it.
The gateway must not create an unlimited number of simultaneous recovery probes.
10. Upstream Requests
When forwarding a request:
- preserve the HTTP method;
- preserve the request path/query;
- forward the request body when present;
- forward the headers required by the gateway contract;
- return the upstream response according to the response contract.
The gateway must not corrupt request bodies during retries or forwarding.
11. Timeouts
Every upstream attempt must have a bounded timeout.
A timeout must:
1. terminate the attempt;
2. record a failed/timeout observation;
3. update backend health;
4. consume the appropriate retry budget;
5. retry only if the request is retryable;
6. otherwise return the deterministic gateway error.
A timed-out backend must not permanently block the gateway.
12. Retry Policy
Retries must be explicitly bounded.
For a request:
attempts <= configured maximum attempts
must always hold.
The gateway must never retry indefinitely.
Retryable failures
The retry policy may include:
- connection failure;
- upstream timeout;
- explicitly configured transient status codes.
Non-retryable requests
Requests that may have side effects must not be blindly retried after the gateway cannot determine whether the upstream processed the request.
The implementation must follow the configured retry safety policy.
Backend selection during retry
When retrying:
- do not select a disabled backend;
- do not select an open-circuit backend;
- prefer another eligible backend when available;
- do not exceed the retry limit.
13. Error Responses
The gateway must produce deterministic responses for at least:
- no matching route;
- no eligible backend;
- upstream connection failure;
- upstream timeout;
- retry exhaustion;
- invalid configuration reload.
The status codes and response format must be consistent for equivalent failures.
The gateway process must remain alive after an upstream failure.
14. Configuration Reload
The running gateway must support configuration reload.
A valid reload must:
1. load the new configuration;
2. validate it completely;
3. make it active atomically;
4. use it for subsequent requests.
An invalid reload must:
1. fail validation;
2. report the failure;
3. leave the previous valid configuration active.
Atomicity
Requests must never observe a partially loaded configuration.
For example, after:
configuration A
      ↓
valid configuration B
new requests use B.
After:
configuration A
      ↓
invalid configuration B
requests continue to use A.
In-flight requests must not be corrupted by a reload.
15. Graceful Shutdown
The gateway must support graceful shutdown.
Its lifecycle is conceptually:
RUNNING
   ↓
DRAINING
   ↓
STOPPED
During draining:
- new requests must be rejected or prevented according to the documented shutdown contract;
- existing requests may finish;
- upstream operations receive a bounded opportunity to finish;
- background health/reload tasks stop;
- the process exits within a bounded timeout.
A forced termination must not be represented as successful graceful shutdown.
16. Concurrency
The gateway must support concurrent requests.
Concurrent traffic must not cause:
- corrupted configuration;
- incorrect backend health state;
- negative/incorrect in-flight counts;
- retry counters to exceed their limits;
- cross-backend state contamination;
- deadlocks;
- indefinite blocking.
Configuration reload may occur concurrently with requests.
Backend failures may occur concurrently with requests.
Shutdown may begin while requests are in flight.
17. Determinism
Equivalent inputs must produce equivalent observable behavior.
Do not make externally visible behavior depend on:
- random map iteration;
- process timing where avoidable;
- uncontrolled filesystem ordering;
- race-dependent state updates.
Adaptive routing may use runtime observations, but the verifier will use sufficient traffic and normalization/tolerance where statistical behavior is involved.
18. Runtime State Bounds
Runtime state must remain bounded.
Do not retain every request forever.
Bound:
- latency samples;
- outcome history;
- request statistics;
- retry metadata;
- recovery records.
The implementation must remain usable after sustained traffic.
19. Required Stateful Workflows
The following workflows are mandatory.
Workflow A — Normal Routing
1. Start the gateway.
2. Configure a route with two healthy backends.
3. Send requests.
4. Verify successful forwarding.
5. Verify backend selection follows the configured policy.
Workflow B — Backend Failure
1. Start with two healthy backends.
2. Send traffic.
3. Make backend A fail.
4. Continue sending requests.
5. Verify A eventually becomes protected.
6. Verify B continues serving traffic.
7. Restore A.
8. Verify A can recover.
Workflow C — Adaptive Latency
1. Start two healthy backends.
2. Give both similar latency.
3. Send traffic.
4. Make A consistently slower.
5. Send additional traffic.
6. Verify routing adapts.
7. Restore A's latency.
8. Verify A can recover its normal routing contribution.
Workflow D — Retry
1. Configure two eligible backends.
2. Cause the first selected backend to fail.
3. Send a retryable request.
4. Verify a retry occurs when permitted.
5. Verify a successful second backend response is returned.
6. Verify attempts never exceed the configured limit.
Workflow E — Timeout
1. Configure an upstream that delays beyond the timeout.
2. Send a request.
3. Verify the attempt terminates.
4. Verify backend state is updated.
5. Verify permitted retry behavior.
6. Verify bounded final response time.
Workflow F — Invalid Reload
1. Start with valid configuration A.
2. Verify A is active.
3. Attempt to load invalid configuration B.
4. Verify the reload fails.
5. Send another request.
6. Verify A is still active.
Workflow G — Graceful Shutdown
1. Start the gateway.
2. Begin an in-flight request.
3. Initiate graceful shutdown.
4. Verify the request follows the shutdown contract.
5. Verify new work is stopped.
6. Verify the process exits within the shutdown bound.
20. Edge Cases
The implementation must handle:
- zero eligible backends;
- all backends failing simultaneously;
- backend recovery while traffic is active;
- repeated configuration reloads;
- invalid configuration after several valid versions;
- reload during an in-flight request;
- shutdown during an upstream timeout;
- concurrent failures;
- retryable upstream errors;
- non-retryable upstream errors;
- duplicate backend IDs;
- zero/negative/invalid weights;
- invalid backend URLs;
- empty route sets;
- exact path versus prefix conflicts;
- method mismatch;
- very slow upstreams;
- upstream connection refusal.
21. Performance
The implementation should remain responsive under concurrent local traffic.
The benchmark should verify bounded behavior rather than rely on fragile machine-specific microbenchmarks.
At minimum:
- startup must complete within a reasonable local bound;
- ordinary routing must remain low-latency;
- hundreds of configured routes must remain searchable without pathological slowdown;
- concurrent requests must not be serialized unnecessarily.
22. Implementation Freedom
Do not assume that the evaluator requires a specific:
- programming language;
- framework;
- web server;
- routing library;
- concurrency model;
- adaptive-routing formula;
- storage mechanism.
Only externally observable behavior is graded.