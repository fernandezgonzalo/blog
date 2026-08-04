---
title: "Circuit Breaker: Zero to Mastery"
subtitle: "Resilience patterns in microservices"
date: 2026-07-27T17:57:04-03:00
# slug: 13fe732
description:
keywords:
weight: 0
categories:
  - draft
collections:
  - draft
tags:
  - draft
summary:
featured_image:
featured_image_preview:
password:
message:
repost:
  enable: false
  url:
mermaid: true
---

## Introduction and Motivation

When a service A depends on a service B over a network (an external API, a database, a microservice), a structural risk emerges: **if B starts failing or responding slowly, A can collapse along with it.**

This happens for a very simple reason: every failed or slow request consumes service A's resources (threads, connections, memory, wait time). If A keeps trying against a down service B, A will eventually exhaust its own resources, propagating the failure upstream in a cascade toward A's clients. This is known as a **cascading failure**.

The **Circuit Breaker** is a design pattern for resilience that cuts off communication with a failing service instead of repeatedly retrying indefinitely. The idea is borrowed directly from electrical engineering: a physical circuit breaker trips and interrupts the circuit when it detects an overload to protect the electrical installation.

Popularized by Michael Nygard in his book *Release It! (2007)*, it is now a standard component in any serious distributed system (Netflix Hystrix, resilience4j, Polly in .NET, pybreaker in Python, Istio at the service mesh layer, etc.).


Pattern Goals:

* Prevent a client from continuously hammering a down service.
* Give the dependent service time to recover.
* Fail fast instead of making the client wait for long timeouts.
* Enable controlled, automatic recovery.


## Theory: The Circuit Breaker Pattern


The Circuit Breaker wraps a "risky" call (typically a network call) and, based on the recent history of successes and failures, decides whether to allow the call to execute or to intercept it immediately by returning an error or a fallback value.


Key Concepts

| Concept | Description |
| ------- | ----------- |
| Failure threshold | The count or percentage of failures that triggers opening the circuit |
| Rolling window (evaluatio window) | The period of time or number of requests over which the failure rate is calculated |
| Open timeout (reset timeout) | The duration the circuit remains open before attempting a probe request |
| Half-open success threshold | The number of consecutive successful calls required to close the crcuit again |
| Fallback | Alternative behavior executed when the circuit is open (default value, cache, queue, graceful error) |


## Why a Simple `try/except` + Retries Is Not Enough

A basic retry mechanism (retrying N times) don't protect the system; in fact, it can worsen the situation by multiplying the load on an already degraded service. The Circuit Breaker pattern is often combined with retries, but it fulfills a distinct purpose:

* **Retry**: Resolves transient, isolated failures.
* **Circuit Breaker**: Resolves sustained failures, preventing repeated calls to a known broken dependency.
* **Timeout**: Prevents a single request from hanging indefinitely.
* **Bulkhead**: Isolates resources (connection pools, threads) so that a failure in one dependency doesn't exhaust shared pools.

In production, these four patterns complement each other rather that acting as substitutes.


## State Machine

At its core, a Circuit Breaker is a finite state machine with three states:

* **CLOSED**: Normal state. Requests pass through. Successes and failures are tracked.
* **OPEN**: The circuit is stripped. Requests are immediately rejected (fail fast) without invoking the underlying service.
* **HALF_OPEN**: After the open timeout elapses, a limited number of probe requests are allowed through to evaluate if the service has recovered.

{{< mermaid >}}
stateDiagram-v2

    [*] --> CLOSED

    CLOSED --> OPEN: Failure threshold reached
    OPEN --> HALF_OPEN: Reset timeout
    HALF_OPEN --> CLOSED: N successes
    HALF_OPEN --> OPEN: Failure

    note right of HALF_OPEN
        Only a limited number of
        requests are allowed.
        Their outcome determines
        whether the circuit closes
        or opens again.
    end note
{{< /mermaid >}}


Transition Details

{{< mermaid >}}
stateDiagram-v2
  direction LR
    CLOSED --> OPEN
{{< /mermaid >}}

Triggers when the rate (or absolute count) of failures within the evaluation window exceeds the configured threshold. Example: 5 consecutive failures, or a 50% failure rate over the last 20 requests.


{{< mermaid >}}
stateDiagram-v2
  direction LR
    OPEN --> HALF_OPEN
{{< /mermaid >}}


Automatic time-based transition. It doesn't depend on inbound requests, after `reset_timeout` seconds, the breaker updates its internal state to permit a test request.

{{< mermaid >}}
stateDiagram-v2
  direction LR
    HALF_OPEN --> CLOSED
{{< /mermaid >}}

If the probe request(s) succeed, the service is assumed to be healthy, and the state resets to normal (**CLOSED**)

{{< mermaid >}}
stateDiagram-v2
  direction LR
    HALF_OPEN --> OPEN
{{< /mermaid >}}

If a probe request fails, the circuit immediately trips open again, resetting the wait timer.


## Request Flowchart

This diagram illustrates how the **Circuit Breaker** evaluates every incoming execution request from application code.

{{< mermaid >}}
flowchart TD
    A([Invoke protected operation]) --> B{Current state?}

    B -- CLOSED --> C[Execute real call]
    C --> D{Result?}
    D -- Success --> E[Record success\nReset failure count]
    D -- Failure --> F[Record failure\nIncrement count]
    F --> G{Failures >= threshold?}
    G -- Yes --> H[Transition to OPEN\nSave opened_at timestamp]
    G -- No --> I[Remain in CLOSED]

    B -- OPEN --> J{Has reset_timeout\nelapsed?}
    J -- No --> K[Reject immediately\nRaise CircuitOpenError\nor execute fallback]
    J -- Yes --> L[Transition to HALF_OPEN]
    L --> M[Allow request\nas probe]

    B -- HALF_OPEN --> N[Execute real call\nwith limits]
    N --> O{Result?}
    O -- Success --> P{Enough consecutive\nsuccesses?}
    P -- Yes --> Q[Transition to CLOSED]
    P -- No --> R[Remain in HALF_OPEN]
    O -- Failure --> S[Transition to OPEN\nReset timestamp]

    E --> Z([Return result to caller])
    H --> Z
    I --> Z
    K --> Z
    M --> Z
    Q --> Z
    R --> Z
    S --> Z
{{< /mermaid >}}


## Secuence Diagram

A concrete scenario: a client application (`SDK`) calling an external endpoint via a `Circuit Breaker` across three different lifecycle phases.


{{< mermaid >}}
sequenceDiagram
    participant Client as Client Code
    participant CB as CircuitBreaker
    participant API as External Service

    Note over CB: Initial State: CLOSED

    Client->>CB: call(get_debts)
    CB->>API: GET /debts/123
    API-->>CB: 200 OK
    CB-->>Client: result
    Note over CB: Success recorded

    Client->>CB: call(get_debts)
    CB->>API: GET /debts/456
    API-->>CB: 500 Error
    CB-->>Client: exception raised
    Note over CB: Failure #1 recorded

    Client->>CB: call(get_debts)
    CB->>API: GET /debts/789
    API-->>CB: timeout
    CB-->>Client: exception raised
    Note over CB: Failure #2. Threshold reached.\nState: OPEN

    Client->>CB: call(get_debts)
    Note over CB: State: OPEN. API call skipped.
    CB-->>Client: CircuitOpenError (fail fast)

    Note over CB: reset_timeout seconds pass

    Client->>CB: call(get_debts)
    Note over CB: State: HALF_OPEN (probe)
    CB->>API: GET /debts/123
    API-->>CB: 200 OK
    CB-->>Client: result
    Note over CB: Probe successful. State: CLOSED
{{< /mermaid >}}


## Implementation from Scratch in Python

Below is a minimal, production-grade reference implementation to understand the internal mechanics before using a third-party library.


```python
import time
import threading
from enum import Enum, auto
from typing import Callable, TypeVar, Any

T = TypeVar("T")


class CircuitState(Enum):
    CLOSED = auto()
    OPEN = auto()
    HALF_OPEN = auto()


class CircuitOpenError(Exception):
    """Raised when the circuit is open and the call is rejected."""


class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        reset_timeout: float = 30.0,
        half_open_success_threshold: int = 2,
        expected_exceptions: tuple[type[Exception], ...] = (Exception,),
    ) -> None:
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.half_open_success_threshold = half_open_success_threshold
        self.expected_exceptions = expected_exceptions

        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._half_open_successes = 0
        self._opened_at: float | None = None
        self._lock = threading.Lock()

    @property
    def state(self) -> CircuitState:
        with self._lock:
            self._maybe_transition_to_half_open()
            return self._state

    def _maybe_transition_to_half_open(self) -> None:
        if (
            self._state is CircuitState.OPEN
            and self._opened_at is not None
            and (time.monotonic() - self._opened_at) >= self.reset_timeout
        ):
            self._state = CircuitState.HALF_OPEN
            self._half_open_successes = 0

    def call(self, func: Callable[..., T], *args: Any, **kwargs: Any) -> T:
        with self._lock:
            self._maybe_transition_to_half_open()
            if self._state is CircuitState.OPEN:
                raise CircuitOpenError(
                    "Circuit open: request rejected (fail fast)"
                )

        try:
            result = func(*args, **kwargs)
        except self.expected_exceptions:
            self._on_failure()
            raise
        else:
            self._on_success()
            return result

    def _on_success(self) -> None:
        with self._lock:
            if self._state is CircuitState.HALF_OPEN:
                self._half_open_successes += 1
                if self._half_open_successes >= self.half_open_success_threshold:
                    self._reset()
            else:
                self._failure_count = 0

    def _on_failure(self) -> None:
        with self._lock:
            if self._state is CircuitState.HALF_OPEN:
                self._trip()
                return

            self._failure_count += 1
            if self._failure_count >= self.failure_threshold:
                self._trip()

    def _trip(self) -> None:
        self._state = CircuitState.OPEN
        self._opened_at = time.monotonic()
        self._failure_count = 0
        self._half_open_successes = 0

    def _reset(self) -> None:
        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._opened_at = None
        self._half_open_successes = 0
```

Basic usage:

```python
breaker = CircuitBreaker(failure_threshold=3, reset_timeout=10.0)

def fetch_debtor_info(tax_id: str) -> dict:
    # Underlying call, e.g., using httpx
    ...

try:
    data = breaker.call(fetch_debtor_info, "20304050607")
except CircuitOpenError:
    # Fallback: cache, default payload, or queue for retry
    data = get_from_cache(tax_id)
```

Key aspects of this implementation:

* **Thread-safe**: Utilizes `Threading.Lock` to guarantee safe state transitions when shared across threads or workers.
* **Single Responsibility**: Explicitly decouples state checks, execution, and state mutations.
* **Targeted Exception Handling**: `expected_exceptions` ensures that domain-level validation errors (eg. `ValueError`) don't count as infrastructure failures.


## Integration with httpx (sync and async)

Modern Python SDKs often support both synchronous and asynchronous operations using `httpx`. Bellow is an async varian of the **Circuit Breaker**.

```python
import asyncio
import time
from enum import Enum, auto


class CircuitState(Enum):
    CLOSED = auto()
    OPEN = auto()
    HALF_OPEN = auto()


class AsyncCircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout: float = 30.0):
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._opened_at: float | None = None
        self._lock = asyncio.Lock()

    async def call(self, coro_func, *args, **kwargs):
        async with self._lock:
            if self._state is CircuitState.OPEN:
                if time.monotonic() - self._opened_at >= self.reset_timeout:
                    self._state = CircuitState.HALF_OPEN
                else:
                    raise CircuitOpenError("Circuit open")

        try:
            result = await coro_func(*args, **kwargs)
        except Exception:
            await self._on_failure()
            raise
        else:
            await self._on_success()
            return result

    async def _on_success(self):
        async with self._lock:
            self._failure_count = 0
            self._state = CircuitState.CLOSED
            self._opened_at = None

    async def _on_failure(self):
        async with self._lock:
            self._failure_count += 1
            if self._failure_count >= self.failure_threshold:
                self._state = CircuitState.OPEN
                self._opened_at = time.monotonic()
```

Integrating with an `httpx.AsyncClient` wrapper by wrapping the low-level HTTP call:

```python
import httpx

class BCRAHttpClient:
    def __init__(self, base_url: str, breaker: AsyncCircuitBreaker):
        self._client = httpx.AsyncClient(base_url=base_url, timeout=5.0)
        self._breaker = breaker

    async def get(self, path: str, **kwargs) -> httpx.Response:
        async def _do_request():
            response = await self._client.get(path, **kwargs)
            response.raise_for_status()
            return response

        return await self._breaker.call(_do_request)
```

This decouples the **Circuit Breaker** from the HTTP engine itself, wrapping the execution closure and making both components independently testable.

## Circuit Breaker Applied to a Real-World SDK (BCRA Case Study)

When building an SDK for a public financial API (suck as the Central Bank of Argentina / BCRA Debtor Database), a **Circuit Breaker** solves specific integration challenges:

* External public APIs are prone to unpredictable outages or latency spikes beyond your control.
* Batch processing operations (eg. iterating over thousands of tax IDs) risk hanging indefinitely on long timeouts without fail-fast guards.
* Isolating breakers per resource endpoint (`Debts`, `BouncedChecks`, `Identity`) prevents partial outages on one route from degrading healthy routes.

Design pattern for resource-level isolation:

```python
class BCRAClient:
    def __init__(self):
        self._breakers = {
            "debts": AsyncCircuitBreaker(failure_threshold=5, reset_timeout=30),
            "bounced_checks": AsyncCircuitBreaker(failure_threshold=5, reset_timeout=30),
            "identity": AsyncCircuitBreaker(failure_threshold=3, reset_timeout=15),
        }

    async def get_debts(self, tax_id: str):
        breaker = self._breakers["debts"]
        return await breaker.call(self._http.get, f"/debts/{tax_id}")
```

If the `bounced_checks` endpoint experiences elevated error rates, its specific breaker trips while `debts` queries continue unaffected.

SDK integration advantages:

* Allows callers to inspect breaker states directly (`client.breaker_state("debts")`) for conditional branching.
* Emits structured telemetry on state transitions without tughtly coupling the SDK to a specific metric collector.
* Empowers the consuming application to define application-specific fallback strategies.


## Existing Libraries

Instead of building custom primitives, production system can leverage mature ecosystem packages:

| Language | Library | Notes |
| --- | --- | --- |
| Python | pybreaker | Traditional, battle-tested standard, supports event listener and pluggable backends (Memory, Redis) |
| Python | Tenacity | Primary focus on retries, but easily combined with custom circuit breaker decorators |
| Python | aiobreaker | An asyncio-native port derived from `pybreaker` |
| Python | purgatory | Async-native circuit breaker featuring built-in Redis support for distributed state storage |
| Python | stamina | Moder retry framework built on top of `tenacity`, featuring native structured logging |
| Go | gobreaker | Lightweight, idiomatic Go implementation with configurable failure thresholds and state transition callbacks |
| Go | go-resiliency | Resilience toolkit providing Circuit Breaker, retries, backoff, and timeout patterns for distributed systems |
| Node.js / TypeScript | opossum | The de facto library, supporting timeouts, fallbacks, health checks, and metrics |
| Node.js / TypeScript | cockatiel | TypeScript-first resilience library implementating Circuit Breaker, Retry, Timeout, Bulkhead, and Fallback policies |
| Java | Resilience4j | Modern, modular fault-tolerance library and the de facto replacement for Hystrix, featuring Circuit Breaker, Retry, Rate Limiter, Bulkhead, and Time Limiter |
| Java | Failsafe | Lightweight resilience library offering Circuit Breaker, Retry, Timeout, Fallback, and asynchronous execution support |


## Advanced Topics

### Rate-Based vs Count-Based Thresholds

An absolute count threshold (eg. 5 failures) can be problematic under low traffic:
5 failures across 5 hours may represent normal network noise rather than a systemic outage. A robust production approach uses a **sliding time window** to evaluate the failure **rate**, similar to `resilience4j`:

* Sliding windows of 60 seconds.
* Minimum throughput threshold before triggering evaluation (prevents false positive on low volume).
* Failure rate defined as a percentage (eg. trip open if failure rate > 50%).

### Expected Exceptions vs System Failures

Not every error constitutes a dependency failure. A `404 Not Found` response on a tax ID query is a valid business response, not an infrastructure issue. Breakers should strictly increment failure metrics on:

* Network transport errors (timeouts, connection refused, DNS resolution failures).
* HTTP `5XX` Server Errors.
* HTTP `429 Too Many Requests` (if configured to protect rate-limit quotas).

### Combining Circuit Breakers, Retries, and Timeouts

The execution wrapping order is critical. Standard layering sequence:

```
Timeout (per-request execution cap)
   → Retry (retry loop with exponential backoff)
       → Circuit Breaker (determines if the retry sequence is permitted)
```

The **Circuit Breaker** wraps the retry mechanism. If the breaker is `OPEN`, short-circuiting prevents initiating entire retry secuences against a degraded dependency.

### Bulkhead Isolation

The **Bulkhead** pattern isolates execution thread pools or connection limits per dependency.
When paired with a **Circuit Breaker**, it ensures that high latency on a single route doesn't saturate the application's global thread pool before the breaker trips.

## Distributed Circuit Breaker

In multi-instance deployments (eg. 10 Kubernetes pods serving a backend service), holding breaker state in local instance memory causes fragmented visibility. Pods independently "discover" that a dependency is down, extending total system recovery time.

Architectural Solutions:

* **Shared State via Redis**: All application nodes track state (failure counters, timestamps) centrally in Redis using atomic operations (`INCR`, `EXPIRE`). Libraries like `purgatory` handle this natively.
* **Service Mesh** / **Sidecar Breaking**: Offload breaker logic to infrastructure proxies (eg. Envoy in Istio/Linkerd), decoupling execution tracking entirely from application runtime code.
* **Eventual Consistency**: In many architectures, local node state isolation is acceptable, independent nodes converge on the `OPEN` state within seconds without adding network hops to state tracking.

{{< mermaid >}}
flowchart LR
    subgraph Cluster["Application Cluster"]
        I1[Instance 1]
        I2[Instance 2]
        I3[Instance 3]
    end

    I1 -- Read/Write State --> R[(Redis:\nShared Breaker\nState)]
    I2 -- Read/Write State --> R
    I3 -- Read/Write State --> R

    I1 -.-> S[Upstream Dependency]
    I2 -.-> S
    I3 -.-> S
{{< /mermaid >}}


## Testing a Circuit Breaker

```python
import pytest
from unittest.mock import Mock

def test_circuit_opens_when_threshold_reached():
    breaker = CircuitBreaker(failure_threshold=3, reset_timeout=100)
    failed_func = Mock(side_effect=ConnectionError("boom"))

    for _ in range(3):
        with pytest.raises(ConnectionError):
            breaker.call(failed_func)

    assert breaker.state == CircuitState.OPEN

    # The 4th call should fail fast without executing failed_func
    with pytest.raises(CircuitOpenError):
        breaker.call(failed_func)

    assert failed_func.call_count == 3  # Execution was intercepted


def test_circuit_transitions_to_half_open_after_timeout(monkeypatch):
    breaker = CircuitBreaker(failure_threshold=1, reset_timeout=5)
    failed_func = Mock(side_effect=ConnectionError())

    with pytest.raises(ConnectionError):
        breaker.call(failed_func)
    assert breaker.state == CircuitState.OPEN

    # Simulate time progression using monotonic clock mocking
    monkeypatch.setattr("time.monotonic", lambda: time.monotonic() + 10)
    assert breaker.state == CircuitState.HALF_OPEN


def test_half_open_resets_to_closed_on_success():
    breaker = CircuitBreaker(
        failure_threshold=1, reset_timeout=0, half_open_success_threshold=1
    )
    func = Mock(side_effect=[ConnectionError(), "ok"])

    with pytest.raises(ConnectionError):
        breaker.call(func)

    result = breaker.call(func)  # Executes in HALF_OPEN state
    assert result == "ok"
    assert breaker.state == CircuitState.CLOSED
```

Testing Guidelines:

* **Avoid real sleep calls** (`time.sleep`): Inject mock clocks or patch `time.monotonic` to keep test execution instantaneous.
* **Assert Call Interception**: Verify that the wrapped target function's execution count stops increasing once the breaker enters the `OPEN` state.
* **Test State Transitions explicitly**: Assert transitions as primary behaviors, not just return payloads.

## Observability and Metrics

An unmonitored **Circuit Breaker** creates dangeours blind spots in production. Expose core metrics to monitoring systems (eg. `Prometheus`:

* `circuit_breaker_state{name="debts"}` (Gauge: 0=CLOSED, 1=OPEN, 2=HALF_OPEN)
* `circuit_breaker_transitions_total{name="debts", from="closed", to="open"}` (Counter)
* `circuit_breaker_rejected_requests_total{name="debts"}` (Counter for intercepted calls)
* `circuit_breaker_failure_rate{name="debts"}` (Gauge)

Structured log payloads should capture transition events with context: breaker name, previous state, new state, trigger condition (threshold breached / timeout elapsed / probe outcome), and timestamp.

## Common Pitfalls and Anti-Patterns

* **Global Monolithic Breakers**: Sharing a single breaker across distinct dependencies (`debts` and `BouncedChecks`) allows failure in one route to block healthy endpoints unnecessarily.
* **Undersized Failure Thresholds**: Settings trip limits too low triggers false positives on transient network noise.
* **Overly Aggressive Reset Timeouts**: Settings short reset timeouts oscillates the breaker between `OPEN` and `HALF_OPEN` without allowing upstream services sufficient time to stabilize.
* **Misclassifying Errors**: Treating expected validation failures (`400 Bad Request`, `404 Not Found`) as system faults trips open circuits on valid business logic errors.
* **Silent Failures**: Failing to publish state change events obscures system health degradation from engineering teams.
* **Thread Safety Neglect**: Omitting locks or atomic primitives leads to race conditions under high concurrent load, corrupting failure counters.

## Conclusion and Next Steps

Building resilient systems isn’t about preventing failures altogether—in distributed architectures, network blips, third-party downtime, and unexpected latency are statistical inevitabilities. Resiliency is about containing blast radiuses and making sure an isolated failure in a downstream dependency doesn't cascade into a complete application outage.

The Circuit Breaker pattern serves as your first line of defense against these cascading failures. By failing fast, shedding load early, and allowing degraded dependencies time to recover, it transforms potentially catastrophic infrastructure incidents into manageable, predictable degradation.

When incorporating Circuit Breakers into your Python services and SDKs, keep these key principles in mind:

1. **Scope your breakers wisely**: Prefer granular, per-resource or per-endpoint breakers over monolithic global instances.
2. **Classify errors correctly**: Distinguish between actual infrastructure/network faults and valid business error codes (like 404 Not Found or 422 Unprocessable Entity).
3. **Combine, don't substitute**: Pair your Circuit Breakers with proper timeouts, exponential backoff retries, and bulkheads for a complete defense-in-depth strategy.
4. **Prioritize observability**: An unmonitored circuit breaker is a silent failure waiting to happen. Always log state transitions and export metrics to Prometheus or your telemetry platform of choice.

By mastering this pattern, you elevate your code from simple API integrations to production-grade, fault-tolerant distributed systems capable of thriving in unpredictable cloud environments.
