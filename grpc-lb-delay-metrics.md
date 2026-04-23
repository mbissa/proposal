Load Balancer Pick Queue Delay Observability
----
* Author(s): Madhav Bissa (@madhavbissa)
* Approver: TBD
* Status: Draft
* Implemented in: Go, Java, C++
* Last updated: 2026-04-22
* Discussion at: TBD

## Abstract

This proposal introduces observability for the duration that RPCs spend waiting in the gRPC client channel waiting for a load balancing pick to complete. Today, when a picker indicates that no subchannel is available, the client channel then defers the RPC (either by queueing it or blocking the caller) until the LB policy provides a new picker or the RPC context is canceled/times out. This latency is invisible to operators, making it difficult to diagnose which layer of the LB policy tree is causing the delay.

This gRFC defines:
1. A new histogram metric (`grpc.lb.pick_queue_duration`) recorded by the client
   channel, with bounded `grpc.lb.queue_reason` and optional `grpc.method`
   labels. The histogram emits one observation per queue reason segment,
   providing per-layer delay breakdown when an RPC transitions through multiple
   LB policy states.
2. A new up-down counter metric (`grpc.lb.pick_queue_active`) tracking the
   number of RPCs currently queued, providing visibility into stuck traffic.
3. An enhancement to the existing "Delayed LB pick complete" span event ([A72])
   to include the queue reason and queue duration as span event attributes.
4. API changes in each language to allow pickers to expose the queue reason as a
   bounded, pre-computed string token, with support for both picker-level tokens
   (uniform policies) and per-pick tokens (RLS, cluster_manager).

## Background

gRPC uses a tree of load balancing policies to select a subchannel for each RPC.
When no suitable subchannel is available, the picker indicates this state to the client channel. The client channel then defers the RPC (either by queueing it or blocking the caller) until the LB policy provides a new picker or the RPC's context is canceled or times out.
Operators currently have no visibility into:
- **How long** RPCs are queued.
- **Why** they are queued (which policy in the tree caused it).

The existing `grpc.client.attempt.duration` metric (A66) includes queue time but
does not isolate it, and provides no information about the queueing cause.

### Related Proposals

- [A66: OpenTelemetry Metrics][A66] — Base instrumentation and stats plugin
  architecture.
- [A72: OpenTelemetry Tracing][A72] — Tracing architecture including the
  existing "Delayed LB pick complete" event.
- [A78: gRPC Metrics for WRR, Pick First, and xDS][A78] — LB policy metrics
  patterns (`grpc.lb.*` naming convention, locality labels).
- [A79: Non-per-call Metrics Architecture][A79] — `GlobalInstrumentsRegistry`,
  `MetricsRecorder`, metric descriptor registration, and stability levels.
- [A91: Outlier Detection Metrics][A91] — Template for LB-level non-per-call
  metrics.
- [A94: Subchannel OTel Metrics][A94] — Subchannel-level metrics.
- [A56: Priority LB Policy][A56] — Priority policy, `initTimer`, failover.
- [A50: xDS Outlier Detection][A50] — Outlier detection ejection behavior.

## Proposal

Using [A79]'s non-per-call metrics architecture and [A72]'s tracing framework,
we will add a histogram metric, an active queue gauge, and enhanced trace span
event that measure the time each RPC spends queued waiting for a load balancing
pick.

Each LB policy's picker provides a bounded string **queue reason token**
describing why it would queue RPCs (e.g., `"pick_first:connecting"`). For
policies with uniform picker state (pick_first, round_robin, ring_hash,
priority), the token is pre-computed at picker construction time. For policies
that make per-request routing decisions (RLS, cluster_manager), the token is
determined inside `Pick()` and varies per-RPC. Delegating policies compose
tokens by prepending their own prefix to their child picker's token (e.g.,
`"priority_p0:pick_first:connecting"`).

The client channel's pick-queueing loop reads this token from the picker when a
pick is queued, records the queue start time, and emits one histogram
observation per queue reason segment — if the token changes during the wait
(e.g., an RLS lookup completes, causing a transition from `rls:lookup_pending`
to `rls:pick_first:connecting`), the current segment is closed and a new one
begins. An up-down counter tracks how many RPCs are currently queued,
providing visibility into stuck traffic that has not yet emitted a histogram
observation.

For policies with uniform picker state, tokens are pre-computed strings stored
on the picker, so the pick path reads them by reference with zero dynamic
allocations. The token API uses optional interfaces (Go) or default-returning
methods (Java, C++) so that existing custom LB policies continue to work without
modification — they simply report an empty token, and the metric is still
recorded under the `grpc.target` label alone.

### Metric Definition

The following metric is registered via the non-per-call metrics framework
defined in [A79][A79].

| Field | Value |
|---|---|
| **Name** | `grpc.lb.pick_queue_duration` |
| **Type** | Float64 Histogram |
| **Unit** | `s` (seconds) |
| **Description** | EXPERIMENTAL. Time an RPC spent queued waiting for a load balancing pick, broken down by the reason for queueing. |
| **Labels** | `grpc.target`, `grpc.lb.queue_reason` |
| **Optional Labels** | `grpc.method` |
| **Bucket Boundaries** | Same as A66 latency buckets: 0, 0.00001, 0.00005, 0.0001, 0.0003, 0.0006, 0.0008, 0.001, 0.002, 0.003, 0.004, 0.005, 0.006, 0.008, 0.01, 0.013, 0.016, 0.02, 0.025, 0.03, 0.04, 0.05, 0.065, 0.08, 0.1, 0.13, 0.16, 0.2, 0.25, 0.3, 0.4, 0.5, 0.65, 0.8, 1, 2, 5, 10, 20, 50, 100 |
| **Default Enabled** | `false` (experimental, opt-in) |

> **Note on histogram aggregation**: The gRPC non-per-call metrics framework
> ([A79]) currently supports only explicit bucket histograms. Explicit buckets
> lose fidelity for values between wide boundaries (e.g., values in the
> `[20, 50)` range are indistinguishable). When the framework adds support for
> exponential bucket histograms, this metric should be migrated to use
> exponential aggregation for better tail-latency fidelity. In the interim,
> operators can override the aggregation strategy to exponential using
> OpenTelemetry SDK Views at the application level.

If the queue reason token changes during the wait (e.g., due to an RLS lookup
completing or a priority failover), the recording loop emits one histogram
observation **per queue reason segment**. A segment ends when a new picker
arrives with a different token, or when the queue wait ends. A single RPC may
therefore contribute multiple histogram observations, each covering the time
spent under a specific queue reason. This per-segment approach gives operators
visibility into how much time each layer of the LB policy tree contributed to
the total queue delay.

The following gauge metric is also registered to provide visibility into RPCs
that are currently queued and have not yet emitted a histogram observation:

| Field | Value |
|---|---|
| **Name** | `grpc.lb.pick_queue_active` |
| **Type** | Int64 UpDownCounter |
| **Unit** | `{call}` |
| **Description** | EXPERIMENTAL. Number of RPCs currently queued waiting for a load balancing pick. |
| **Labels** | `grpc.target` |
| **Default Enabled** | `false` (experimental, opt-in) |

The counter is incremented (`+1`) when a pick is first queued and decremented
(`-1`) when the queue wait ends (successful pick or cancellation). This matches
the pattern used by `grpc.subchannel.open_connections` ([A94]) and
`grpc.tcp.connection_count` ([A80]).

#### Label Definitions

| Label | Type | Description |
|---|---|---|
| `grpc.target` | String | The target URI of the channel. Required. |
| `grpc.lb.queue_reason` | String | The bounded queue reason token from the picker. Optional label. |
| `grpc.method` | String | The full method name of the RPC (e.g., `/pkg.Service/Method`). Optional label. Available from `PickInfo` at queue time. Primarily useful for policies like RLS where queue behavior varies by method. |

### Queue Reason Token Semantics

Queue reason tokens are static, bounded strings following the grammar:

```
token       = terminal | prefix ":" token
prefix      = policy_prefix
terminal    = policy_name ":" state
policy_name = "pick_first" | "round_robin" | "ring_hash" | "rls" | "cds"
```

Dynamic values (IP addresses, cluster names, endpoint identifiers) are
**strictly forbidden** in tokens.

#### Terminal Tokens

Terminal tokens are emitted by leaf policies when they cannot complete a pick.

| Policy | Token | When Emitted |
|---|---|---|
| `pick_first` | `pick_first:connecting` | No READY subchannel; connection attempt in progress. |
| `round_robin` | `round_robin:connecting` | No READY subchannel in the round-robin set. |
| `ring_hash` | `ring_hash:connecting` | The hashed (or scanned) endpoint is in CONNECTING or IDLE state. |
| `rls` | `rls:lookup_pending` | Waiting for an RLS server response for the request key. |
| `rls` | `rls:throttled` | RLS requests are being throttled due to server errors. |
| `cds` | `cds:discovery_pending` | Waiting for the xDS CDS resource. |

#### Delegating Prefixes

Delegating policies prepend a prefix to their child picker's token.

| Policy | Prefix | Notes |
|---|---|---|
| `priority` | `priority_p{N}:` | N is the 0-based priority index. Typical deployments use ≤5 priorities, bounding cardinality. |
| `cluster_manager` | *(per-pick)* | Routes RPCs to different child pickers per-request. The token is the child picker's token, resolved at pick time. See Per-Pick Tokens. |
| `rls` | `rls:` | When routing to a resolved child policy (post-lookup). |
| `cds` | `cds:` | When delegating to a child policy (post-discovery). |
| `wrr_locality` | *(transparent)* | Passes child tokens through unmodified. |
| `outlier_detection` | *(transparent)* | Passes child tokens through unmodified; OD does not itself cause queueing. |

#### Token Composition

For most policies, token composition occurs at **picker construction time**, not
on the pick path. When a delegating policy creates its picker, it reads the
child picker's token and stores the composite string as a member of its own
picker. This ensures zero allocations during `Pick()`.

**Example flow** for `priority → pick_first`:
1. `pick_first` creates a picker with token `"pick_first:connecting"`.
2. `priority` wraps the child, creating its picker with token
   `"priority_p0:pick_first:connecting"` (stored as a member string).

When a delegating policy has a **READY** child (no queueing), the token is the
empty string `""`, and the pick completes without queueing.

#### Per-Pick Tokens

Some policies make per-request routing decisions within `Pick()`, causing the
queue reason to vary across RPCs handled by the same picker. These policies
provide their token via the queue result rather than a picker-level method:

- **RLS**: A single `rlsPicker` performs a per-request cache lookup. A cache
  miss queues with `rls:lookup_pending`, while a cache hit that delegates to a
  child in CONNECTING state queues with `rls:pick_first:connecting`. The token
  must be determined inside `Pick()` and attached to the queue result.

- **cluster_manager**: Routes RPCs to different child pickers based on the
  cluster selected by xDS routing. Different RPCs may reach different child
  pickers in different states. The token is read from whichever child picker
  handled the specific RPC.

The mechanism for attaching per-pick tokens is described in the language-specific
API sections below.

### Language-Specific API Changes

#### Go

Go's `Picker` interface has a single `Pick()` method. Adding a method would
break all existing implementations. Two mechanisms are provided for supplying
queue reason tokens:

**Picker-level token** — for policies with uniform picker state (pick_first,
round_robin, ring_hash, priority). An optional interface that pickers may
implement:

```go
// In package balancer

// QueueMetricTokener is an optional interface that Pickers can implement
// to provide a bounded metric token describing why this picker queues RPCs.
// The token MUST be a static, pre-computed string with no dynamic components.
type QueueMetricTokener interface {
    // QueueMetricToken returns the bounded queue reason token.
    // Returns "" if the picker does not queue RPCs.
    QueueMetricToken() string
}
```

**Per-pick token** — for policies where the queue reason varies per-RPC (RLS,
cluster_manager). A structured error type that wraps `ErrNoSubConnAvailable`
with a token:

```go
// In package balancer

// QueueError is returned by Pick() to signal queueing with a per-pick
// metric token. It is equivalent to ErrNoSubConnAvailable but carries
// a queue reason token that may vary per-RPC.
type QueueError struct {
    // MetricToken is the bounded queue reason token for this specific pick.
    MetricToken string
}

func (e *QueueError) Error() string { return "no SubConn available" }
func (e *QueueError) Is(target error) bool {
    return target == ErrNoSubConnAvailable
}
```

The recording loop resolves the token with the following precedence: if the
error is a `*QueueError`, use its `MetricToken`; otherwise, check whether the
picker implements `QueueMetricTokener`; otherwise, use the empty string.

Built-in pickers (`pick_first`, `round_robin`, `ring_hash`, `priority`, etc.)
implement `QueueMetricTokener`. Custom LB policies that implement neither
mechanism report an empty token.

**Example — `pick_first` (picker-level):**
```go
type pfPicker struct {
    subConn    balancer.SubConn
    queueToken string // set at construction: "pick_first:connecting" or ""
}

func (p *pfPicker) Pick(balancer.PickInfo) (balancer.PickResult, error) {
    return balancer.PickResult{SubConn: p.subConn}, nil
}

func (p *pfPicker) QueueMetricToken() string {
    return p.queueToken
}
```

**Example — `rls` (per-pick):**
```go
func (p *rlsPicker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
    // ... cache lookup using info.FullMethodName and request keys ...
    switch {
    case dcEntry == nil && pendingEntry == nil:
        p.sendRouteLookupRequest(cacheKey, ...)
        return balancer.PickResult{},
            &balancer.QueueError{MetricToken: "rls:lookup_pending"}
    case dcEntry == nil && pendingEntry != nil:
        return balancer.PickResult{},
            &balancer.QueueError{MetricToken: "rls:lookup_pending"}
    case dcEntry != nil:
        // Delegate to child policy — child's Pick() may return its
        // own QueueError or ErrNoSubConnAvailable.
        pr, err := childPicker.Pick(info)
        if qe, ok := err.(*balancer.QueueError); ok {
            qe.MetricToken = "rls:" + qe.MetricToken
        }
        return pr, err
    }
}
```

**Example — `priority` (picker-level, delegating):**
```go
type priorityPicker struct {
    childPicker balancer.Picker
    queueToken  string // e.g., "priority_p0:" + childToken
}

func (p *priorityPicker) QueueMetricToken() string {
    return p.queueToken
}
```

#### Java

Java's `PickResult` is `final` and `@Immutable`. The per-pick token is carried
on the `PickResult` itself via a new factory method, which naturally supports
policies like RLS where the queue reason varies per-RPC. The `SubchannelPicker`
also gets a default method for delegating policies that have uniform state:

```java
// In LoadBalancer.PickResult

// New field (private, immutable)
@Nullable private final String queueMetricToken;

// New factory method for queue results with a reason token
public static PickResult withNoResult(String queueMetricToken) {
    return new PickResult(null, null, Status.OK, false, null, queueMetricToken);
}

// Getter
public String getQueueMetricToken() {
    return queueMetricToken != null ? queueMetricToken : "";
}
```

The existing `withNoResult()` continues to return the `NO_RESULT` singleton with
an empty token. The `SubchannelPicker` also gets a default method for
delegating policies to read the child's token:

```java
// In LoadBalancer.SubchannelPicker

/**
 * Returns the bounded metric token describing why this picker queues RPCs.
 * Default returns empty string (no queueing information).
 */
public String getQueueMetricToken() { return ""; }
```

**Example implementation in `PickFirstLeafLoadBalancer`:**
```java
private class Picker extends SubchannelPicker {
    private final String queueMetricToken;

    Picker(String token) { this.queueMetricToken = token; }

    @Override
    public PickResult pickSubchannel(PickSubchannelArgs args) {
        // When queueing, use the token-aware factory:
        return PickResult.withNoResult(queueMetricToken);
    }

    @Override
    public String getQueueMetricToken() { return queueMetricToken; }
}
```

#### C++ (Core)

The `PickResult::Queue` struct is extended to hold a `string_view` carrying the
per-pick token. Since `Queue` is returned from `Pick()`, this naturally supports
policies where the token varies per-RPC. The `string_view` points to a
picker-owned string, involving zero dynamic allocations on the pick path:

```cpp
// In LoadBalancingPolicy::PickResult

struct Queue {
    // Bounded queue reason token. Points to a string owned by the Picker.
    // The Picker's lifetime exceeds the pick, so this is safe.
    // Empty string_view means no token is provided.
    absl::string_view metric_token;

    Queue() = default;
    explicit Queue(absl::string_view token) : metric_token(token) {}
};
```

The `SubchannelPicker` base class gets a virtual method for delegating policies:

```cpp
class SubchannelPicker : public DualRefCounted<SubchannelPicker> {
 public:
    virtual PickResult Pick(PickArgs args) = 0;

    // Returns the bounded queue reason token. Default returns empty.
    virtual absl::string_view GetQueueMetricToken() const { return ""; }
};
```

**Example implementation in a leaf picker:**
```cpp
class PickFirstQueuePicker final : public SubchannelPicker {
 public:
    PickResult Pick(PickArgs) override {
        return PickResult::Queue(kToken);
    }
    absl::string_view GetQueueMetricToken() const override { return kToken; }

 private:
    static constexpr absl::string_view kToken = "pick_first:connecting";
};
```

**Example delegating picker:**
```cpp
class PriorityPicker final : public SubchannelPicker {
 public:
    PriorityPicker(int priority_index,
                   RefCountedPtr<SubchannelPicker> child)
        : child_(std::move(child)) {
        // Compose token at construction time (stored as member string).
        auto child_token = child_->GetQueueMetricToken();
        if (!child_token.empty()) {
            composite_token_ = absl::StrCat("priority_p",
                                             priority_index, ":",
                                             child_token);
        }
    }

    PickResult Pick(PickArgs args) override {
        auto result = child_->Pick(args);
        if (auto* q = absl::get_if<PickResult::Queue>(&result.result)) {
            q->metric_token = composite_token_;
        }
        return result;
    }

    absl::string_view GetQueueMetricToken() const override {
        return composite_token_;
    }

 private:
    RefCountedPtr<SubchannelPicker> child_;
    std::string composite_token_;  // owned by this picker
};
```

### Language-Specific Recording Logic

Recording is performed by the client channel's pick-queueing infrastructure, not
by the LB policy itself. The LB policy provides the token; the channel measures
time and records the metric.

#### Go — `picker_wrapper.go`

In the `pick()` method's blocking loop, the recording logic resolves the token
from either a `QueueError` or the `QueueMetricTokener` interface, and emits a
histogram observation each time the token changes (per-segment emission):

```go
func (pw *pickerWrapper) pick(ctx context.Context, failfast bool,
    info balancer.PickInfo) (pick, error) {
    var queueStartTime time.Time
    var queueToken string
    var queued bool
    method := info.FullMethodName

    for {
        // ... existing picker load and blocking logic ...

        pickResult, err := p.Pick(info)
        if err != nil {
            if errors.Is(err, balancer.ErrNoSubConnAvailable) {
                // Resolve per-pick token from QueueError, or fall
                // back to picker-level QueueMetricTokener.
                currentToken := ""
                if qe, ok := err.(*balancer.QueueError); ok {
                    currentToken = qe.MetricToken
                } else if qt, ok := p.(balancer.QueueMetricTokener); ok {
                    currentToken = qt.QueueMetricToken()
                }

                if !queued {
                    // First queue event — start timing, increment counter.
                    queued = true
                    queueStartTime = time.Now()
                    queueToken = currentToken
                    pickQueueActiveMetric.Record(metricsRecorder, 1, target)
                } else if currentToken != queueToken {
                    // Token changed (new picker with different state).
                    // Emit segment for the previous token.
                    duration := time.Since(queueStartTime).Seconds()
                    pickQueueDurationMetric.Record(metricsRecorder,
                        duration, target, queueToken, method)
                    // Start new segment.
                    queueStartTime = time.Now()
                    queueToken = currentToken
                }
                continue
            }
            // ... existing error handling ...
        }

        // Pick succeeded. If we were queued, record final segment.
        if queued {
            duration := time.Since(queueStartTime).Seconds()
            pickQueueDurationMetric.Record(metricsRecorder,
                duration, target, queueToken, method)
            pickQueueActiveMetric.Record(metricsRecorder, -1, target)
            if attemptSpan != nil {
                attemptSpan.AddEvent("Delayed LB pick complete",
                    attribute.Float64("queue_duration", duration),
                    attribute.String("queue_reason", queueToken))
            }
        }
        // ... existing transport handling ...
    }
}
```

On context cancellation (existing `case <-ctx.Done():` block), the same
recording logic applies:
```go
case <-ctx.Done():
    if queued {
        duration := time.Since(queueStartTime).Seconds()
        pickQueueDurationMetric.Record(metricsRecorder,
            duration, target, queueToken, method)
        pickQueueActiveMetric.Record(metricsRecorder, -1, target)
        if attemptSpan != nil {
            attemptSpan.AddEvent("Delayed LB pick complete",
                attribute.Float64("queue_duration", duration),
                attribute.String("queue_reason", queueToken))
        }
    }
    // ... existing error return ...
```

The `metricsRecorder` and `target` are available from the `ClientConn` that owns
the `pickerWrapper`.

#### Java — `DelayedClientTransport.java`

When a `PendingStream` is created in `createPendingStream()`, the queue start
time and token are captured:

```java
private PendingStream createPendingStream(PickSubchannelArgs args,
    ClientStreamTracer[] tracers, PickResult pickResult) {
    PendingStream pendingStream = new PendingStream(args, tracers);
    pendingStream.queueStartNanos = System.nanoTime();
    pendingStream.queueToken = (pickerState.lastPicker != null)
        ? pickerState.lastPicker.getQueueMetricToken() : "";
    // ... existing logic ...
}
```

When the stream is dequeued in `reprocess()` (transport obtained), or cancelled,
the duration is recorded:

```java
if (transport != null) {
    long durationNanos = System.nanoTime() - stream.queueStartNanos;
    double durationSeconds = durationNanos / 1_000_000_000.0;
    metricsRecorder.recordPickQueueDuration(durationSeconds,
        target, stream.queueToken);
    stream.getTracer().recordAnnotation("Delayed LB pick complete",
        Attributes.of(
            "queue_duration", durationSeconds,
            "queue_reason", stream.queueToken));
    // ... existing stream creation ...
}
```

#### C++ (Core) — `load_balanced_call_destination.cc`

In the pick loop, when `PickResult::Queue` is returned, the start time and token
are captured. When the pick eventually succeeds or the call is cancelled, the
duration is recorded:

```cpp
// In the pick processing lambda (simplified):
auto queue_func = [&](LoadBalancingPolicy::PickResult::Queue* queue_pick) {
    if (!queue_start_time.has_value()) {
        queue_start_time = Timestamp::Now();
        queue_token = std::string(queue_pick->metric_token);
    }
    return Continue{};
};

// On successful pick (in the completion callback):
if (queue_start_time.has_value()) {
    Duration queue_duration = Timestamp::Now() - *queue_start_time;
    stats_plugin_group.RecordHistogram(
        kPickQueueDurationHandle,
        queue_duration.seconds(),
        {target}, {queue_token});
    call_tracer->RecordAnnotation("Delayed LB pick complete", {
        {"queue_duration", absl::StrCat(queue_duration.seconds())},
        {"queue_reason", queue_token}
    });
}
```

#### C-Core Specifics: RLS Cache Misses and HTTP/2 Max Concurrent Streams

In `grpc-core`, the boundary between Load Balancing and the Transport layer introduces two highly specific queuing scenarios that must be handled distinctly in the `PickResult::Queue` tokenization.

**1. RLS Cache Misses (Dynamic Token Lifetime)**
Unlike static policies (like `pick_first`), the RLS policy in C-Core evaluates targets dynamically per-call. When a cache miss occurs, the RLS policy initiates a control-plane request and returns `PickResult::Queue`.
*   **The C-Core Constraint:** The `PickResult::Queue` struct uses an `absl::string_view` to prevent allocations on the hot path. However, because an RLS cache miss is dynamic, the string it points to must safely outlive the pick.
*   **Implementation:** The RLS LB Policy must maintain a static, pre-allocated pool of `std::string` constants for its state transitions (e.g., `static const std::string kRlsPending = "rls:lookup_pending";`). During a cache miss `Pick()`, the policy returns a `string_view` pointing explicitly to this constant memory address, ensuring safe reference lifecycle without dynamic memory allocation per RPC.

**2. MAX_CONCURRENT_STREAMS (Transport-Level Queueing)**
In C-Core, queueing can occur even when the LB Policy successfully finds a `READY` subchannel. If the HTTP/2 transport for that subchannel has reached its `SETTINGS_MAX_CONCURRENT_STREAMS` limit, the `grpc_call` must be queued.
*   **The Distinction:** This is a *transport* queue, not an *LB routing* queue.
*   **Implementation:** The LB policy's `Pick()` method will actually succeed and return a `READY` subchannel. However, the `client_channel` filter will detect the transport exhaustion. To maintain observability, the `client_channel` filter itself will inject a synthetic token: `"transport:max_concurrent_streams"`. The queue timer will start, and the RPC will wait in the filter's pending list until a stream becomes available or the context deadline is exceeded. This clearly separates network capacity limits from LB routing failures in the resulting telemetry.

### Metric Instrument Registration

#### Go

```go
import estats "google.golang.org/grpc/experimental/stats"

var pickQueueDurationMetric = estats.RegisterFloat64Histo(
    estats.MetricDescriptor{
        Name:           "grpc.lb.pick_queue_duration",
        Description:    "EXPERIMENTAL. Time an RPC spent queued waiting " +
                        "for a load balancing pick.",
        Unit:            "s",
        Labels:          []string{"grpc.target", "grpc.lb.queue_reason"},
        OptionalLabels: []string{"grpc.method"},
        Default:        false,
    })

var pickQueueActiveMetric = estats.RegisterInt64UpDownCount(
    estats.MetricDescriptor{
        Name:           "grpc.lb.pick_queue_active",
        Description:    "EXPERIMENTAL. Number of RPCs currently queued " +
                        "waiting for a load balancing pick.",
        Unit:           "{call}",
        Labels:         []string{"grpc.target"},
        Default:        false,
    })
```

#### Java

```java
private static final LongHistogramInstrumentDescriptor PICK_QUEUE_DURATION =
    InstrumentRegistry.registerDoubleHistogram(
        "grpc.lb.pick_queue_duration",
        "EXPERIMENTAL. Time an RPC spent queued waiting "
            + "for a load balancing pick.",
        "s",
        List.of("grpc.target", "grpc.lb.queue_reason"),  // required labels
        List.of("grpc.method"),                           // optional labels
        false);                                          // default disabled

private static final LongUpDownCounterMetricInstrument PICK_QUEUE_ACTIVE =
    InstrumentRegistry.registerLongUpDownCounter(
        "grpc.lb.pick_queue_active",
        "EXPERIMENTAL. Number of RPCs currently queued "
            + "waiting for a load balancing pick.",
        "{call}",
        List.of("grpc.target"),
        List.of(),
        false);
```

#### C++ (Core)

```cpp
const auto kPickQueueDurationHandle =
    GlobalInstrumentsRegistry::RegisterDoubleHistogram(
        "grpc.lb.pick_queue_duration",
        "EXPERIMENTAL. Time an RPC spent queued waiting "
        "for a load balancing pick.",
        "s",
        /*label_keys=*/{"grpc.target", "grpc.lb.queue_reason"},
        /*optional_label_keys=*/{"grpc.method"},
        /*enable_by_default=*/false);

const auto kPickQueueActiveHandle =
    GlobalInstrumentsRegistry::RegisterUpDownInt64Counter(
        "grpc.lb.pick_queue_active",
        "EXPERIMENTAL. Number of RPCs currently queued "
        "waiting for a load balancing pick.",
        "{call}",
        /*label_keys=*/{"grpc.target"},
        /*optional_label_keys=*/{},
        /*enable_by_default=*/false);
```

### Metric Stability

Per [A79][A79], this metric starts as **experimental** and **disabled by
default**. Users must explicitly opt in via their OpenTelemetry plugin
configuration. The metric description is prefixed with `EXPERIMENTAL.` to
signal this status.

The metric will be promoted to stable after it has been implemented and validated
in all three languages, with at least one release cycle of user feedback.

### Temporary environment variable protection

The metric will be guarded by the environment variable
`GRPC_EXPERIMENTAL_ENABLE_PICK_QUEUE_METRIC`. When set to `true`, the metric
will be registered and reported even if the user has not explicitly enabled it
via the OpenTelemetry plugin configuration. This guard will be removed once the
feature is deemed stable.

### Tracing Enhancement

[A72] defines a "Delayed LB pick complete" span event on the attempt span,
emitted when an RPC experiences load balancer pick delay. Today, this event
carries no attributes. This proposal enhances it with the following attributes:

| Attribute Key | Type | Description |
|---|---|---|
| `queue_duration` | double | The queue wait duration in seconds. |
| `queue_reason` | string | The queue reason token from the picker. |

The event is emitted at the point where the queue wait ends (successful pick or
cancellation), on the attempt span. The attribute values come from the same
queue start time and token already captured for the metric.

This enhancement is additive to [A72] — the event name remains "Delayed LB pick
complete" and the event is still emitted only when the RPC experienced queueing.
The only change is the addition of attributes. If tracing is not configured via
the OpenTelemetry plugin, no span events are emitted and there is no overhead.

## Rationale

The metric is recorded by the client channel's pick-queueing infrastructure
rather than by the LB policy itself, because the LB policy knows its internal
state but does not know how many RPCs are waiting or for how long. Only the
channel physically buffers RPCs and can accurately measure per-RPC queue
duration. This mirrors the existing design where per-call metrics ([A66]) are
recorded by the channel, not by LB policies.

Two mechanisms are provided for supplying queue reason tokens — a picker-level
optional interface (`QueueMetricTokener`) and a per-pick error/result type
(`QueueError` / `PickResult.withNoResult(token)` / `Queue::metric_token`). Most
LB policies have uniform picker state: all RPCs see the same queue reason from a
given picker instance (pick_first, round_robin, ring_hash, priority). These
policies use the picker-level mechanism, which involves zero allocations on the
pick path. However, RLS and cluster_manager make per-request routing decisions
within `Pick()` — RLS does a per-request cache lookup, and cluster_manager
routes RPCs to different child pickers based on the xDS-selected cluster. For
these policies, the queue reason varies per-RPC, requiring the token to be
determined inside `Pick()` and attached to the queue result.

The recording loop emits one histogram observation per queue reason segment
rather than a single observation per RPC. An RPC that transitions through
multiple queue reasons (e.g., `rls:lookup_pending` for 10 seconds, then
`rls:pick_first:connecting` for 0.1 seconds after the lookup completes) produces
two histogram observations, one for each segment. This per-segment approach
provides visibility into how much time each layer of the LB policy tree
contributed to the total queue delay. Without it, a single 10.1-second
observation attributed to `rls:lookup_pending` would obscure the fact that the
RLS lookup accounted for the vast majority of the delay.

The `grpc.lb.pick_queue_active` up-down counter is provided because the
histogram metric is emitted only when a queue wait ends. RPCs that are
indefinitely stuck in the queue (e.g., target unreachable, infinite backoff)
never emit a histogram observation. The counter gives operators a real-time
count of currently-queued RPCs, enabling alerting on stuck traffic. Every LB
policy in the tree can cause indefinite queueing (pick_first cycling through
CONNECTING/TF, RLS server unreachable, CDS resource never arriving), making this
counter essential. An up-down counter is used rather than a callback gauge
because the recording loop has the exact synchronous entry/exit points, matching
the pattern used by `grpc.subchannel.open_connections` ([A94]) and
`grpc.tcp.connection_count` ([A80]).

`grpc.method` is included as an optional label because some policies (RLS,
cluster_manager) make routing decisions based on the method name, causing queue
behavior to vary across methods. For simpler policies (pick_first,
round_robin), the queue behavior is method-independent, but operators may still
want per-method queue analysis to correlate with per-call attempt metrics. As an
optional label, it defaults to off and only adds cardinality when explicitly
enabled.

`wrr_locality` and `outlier_detection` are treated as transparent because
neither makes independent queueing decisions. `wrr_locality` is a configuration
wrapper; `outlier_detection` manipulates subchannel state but the actual
queueing is decided by the child policy's picker. When outlier detection ejects
100% of endpoints, the child policy (e.g., pick_first) enters
TRANSIENT_FAILURE and returns an error rather than a queue result, so no queue
metric is recorded. In partial ejection scenarios, the child is genuinely in
CONNECTING state and the token is accurate. Operators can correlate queue
duration with the existing `grpc.lb.outlier_detection.ejections_enforced` metric
([A91]) for root cause analysis.

The tracing enhancement reuses the same bounded queue reason token as the metric
label rather than introducing a separate high-cardinality trace reason string.
This keeps the API surface minimal — one token serves both metrics and traces —
and avoids the need for pickers to maintain two separate strings. If richer,
dynamic trace context (e.g., IP addresses, RLS keys) is needed in the future,
it can be added as additional span event attributes without changing the token
API.

## Implementation

Implementation will proceed in Go, Java, and C++ (Core), in that order.

[A66]: A66-otel-stats.md
[A72]: A72-open-telemetry-tracing.md
[A78]: A78-grpc-metrics-wrr-pf-xds.md
[A79]: A79-non-per-call-metrics-architecture.md
[A80]: A80-tcp-metrics.md
[A91]: A91-outlier-detection-metrics.md
[A94]: A94-subchannel-otel-metrics.md
[A56]: A56-priority-lb-policy.md
[A50]: A50-xds-outlier-detection.md