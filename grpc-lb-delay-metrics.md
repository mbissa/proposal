Load Balancer Pick Queue Delay Observability
----
* Author(s): Madhav Bissa (@madhavbissa)
* Approver: TBD
* Status: Draft
* Implemented in: Go, Java, C++
* Last updated: 2026-04-21
* Discussion at: TBD

## Abstract

This proposal introduces observability for the duration that RPCs spend queued
in the gRPC client channel waiting for a load balancing pick to complete. Today,
when a picker returns "queue" (e.g., because subchannels are in `CONNECTING`
state), the client channel silently buffers the RPC until a new picker is
available. This latency is invisible to operators, making it difficult to
diagnose which layer of the LB policy tree is causing delay.

This gRFC defines:
1. A new histogram metric (`grpc.lb.pick_queue_duration`) recorded by the client
   channel, with a bounded `grpc.lb.queue_reason` label whose value is supplied
   by the LB policy's picker.
2. An enhancement to the existing "Delayed LB pick complete" span event ([A72])
   to include the queue reason and queue duration as span event attributes.
3. API changes in each language to allow pickers to expose the queue reason as a
   bounded, pre-computed string token.

## Background

gRPC uses a tree of load balancing policies to select a subchannel for each RPC.
When no suitable subchannel is available, the picker returns a "queue" result and
the client channel buffers the RPC until the LB policy provides a new picker.
This buffering can add significant latency, but operators currently have no
visibility into:
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
we will add a histogram metric and enhanced trace span event that measure the
time each RPC spends queued waiting for a load balancing pick.

Each LB policy's picker will pre-compute a bounded string **queue reason token**
at construction time describing why it would queue RPCs (e.g.,
`"pick_first:connecting"`). Delegating policies compose tokens by prepending
their own prefix to their child picker's token (e.g.,
`"priority_p0:pick_first:connecting"`). The client channel's pick-queueing loop
reads this token from the picker when a pick is queued, records the queue start
time, and when the queue wait ends (either by a successful pick or context
cancellation), records both a histogram observation and a span event with
attributes.

Tokens are pre-computed strings stored on the picker, so the pick path reads
them by reference with zero dynamic allocations. The token API uses optional
interfaces (Go) or default-returning methods (Java, C++) so that existing custom
LB policies continue to work without modification — they simply report an empty
token, and the metric is still recorded under the `grpc.target` label alone.

### Metric Definition

The following metric is registered via the non-per-call metrics framework
defined in [A79][A79].

| Field | Value |
|---|---|
| **Name** | `grpc.lb.pick_queue_duration` |
| **Type** | Float64 Histogram |
| **Unit** | `s` (seconds) |
| **Description** | EXPERIMENTAL. Time an RPC spent queued waiting for a load balancing pick, broken down by the reason for queueing. |
| **Labels** | `grpc.target` |
| **Optional Labels** | `grpc.lb.queue_reason` |
| **Bucket Boundaries** | Same as A66 latency buckets: 0, 0.00001, 0.00005, 0.0001, 0.0003, 0.0006, 0.0008, 0.001, 0.002, 0.003, 0.004, 0.005, 0.006, 0.008, 0.01, 0.013, 0.016, 0.02, 0.025, 0.03, 0.04, 0.05, 0.065, 0.08, 0.1, 0.13, 0.16, 0.2, 0.25, 0.3, 0.4, 0.5, 0.65, 0.8, 1, 2, 5, 10, 20, 50, 100 |
| **Default Enabled** | `false` (experimental, opt-in) |

The metric is recorded once per RPC that experienced queueing, when the queue
wait ends. The duration is the wall-clock time from when the pick was first
queued to when a successful pick was obtained (or the RPC was cancelled). If the
queue reason token changes during the wait (e.g., due to priority failover), the
final token observed at the time of dequeue is used as the label value.

#### Label Definitions

| Label | Type | Description |
|---|---|---|
| `grpc.target` | String | The target URI of the channel. Required. |
| `grpc.lb.queue_reason` | String | The bounded queue reason token from the picker. Optional label. |

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
| `cluster_manager` | `cluster_manager:` | Cluster names are intentionally omitted for cardinality safety. |
| `rls` | `rls:` | When routing to a resolved child policy (post-lookup). |
| `cds` | `cds:` | When delegating to a child policy (post-discovery). |
| `wrr_locality` | *(transparent)* | Passes child tokens through unmodified. |
| `outlier_detection` | *(transparent)* | Passes child tokens through unmodified; OD does not itself cause queueing. |

#### Token Composition

Token composition occurs at **picker construction time**, not on the pick path.
When a delegating policy creates its picker, it reads the child picker's token
and stores the composite string as a member of its own picker. This ensures
zero allocations during `Pick()`.

**Example flow** for `cluster_manager → priority → pick_first`:
1. `pick_first` creates a picker with token `"pick_first:connecting"`.
2. `priority` wraps the child, creating its picker with token
   `"priority_p0:pick_first:connecting"` (stored as a member string).
3. `cluster_manager` wraps priority, creating its picker with token
   `"cluster_manager:priority_p0:pick_first:connecting"`.

When a delegating policy has a **READY** child (no queueing), the token is the
empty string `""`, and the pick completes without queueing.

### Language-Specific API Changes

#### Go

Go's `Picker` interface has a single `Pick()` method. Adding a method would
break all existing implementations. Instead, we define an **optional interface**
that pickers may implement:

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

Built-in pickers (`QueuePicker`, `errPicker`, etc.) and LB policy pickers
(`pick_first`, `round_robin`, `ring_hash`, `priority`, etc.) will implement this
interface. Custom LB policies that do not implement it will simply report an
empty token, and the metric will still record the queue duration under the
`grpc.target` label alone.

**Example implementation in `pick_first`:**
```go
type pfPicker struct {
    subConn  balancer.SubConn
    queueToken string // set at construction: "pick_first:connecting" or ""
}

func (p *pfPicker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
    return balancer.PickResult{SubConn: p.subConn}, nil
}

func (p *pfPicker) QueueMetricToken() string {
    return p.queueToken
}
```

**Example delegating implementation in `priority`:**
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

Java's `PickResult` is `final` and `@Immutable`. We add a new static factory
method and a getter, which is backwards-compatible:

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

The `PickResult::Queue` struct is extended to hold a `string_view` pointing to
a picker-owned string. This involves zero dynamic allocations on the pick path
since `string_view` is a non-owning reference:

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

In the `pick()` method's blocking loop:

```go
func (pw *pickerWrapper) pick(ctx context.Context, failfast bool,
    info balancer.PickInfo) (pick, error) {
    var queueStartTime time.Time
    var queueToken string

    for {
        // ... existing picker load and blocking logic ...

        pickResult, err := p.Pick(info)
        if err != nil {
            if err == balancer.ErrNoSubConnAvailable {
                // Record queue start on first queue event.
                if queueStartTime.IsZero() {
                    queueStartTime = time.Now()
                    if qt, ok := p.(balancer.QueueMetricTokener); ok {
                        queueToken = qt.QueueMetricToken()
                    }
                }
                continue
            }
            // ... existing error handling ...
        }

        // Pick succeeded. If we were queued, record the duration and span event.
        if !queueStartTime.IsZero() {
            duration := time.Since(queueStartTime).Seconds()
            pickQueueDurationMetric.Record(metricsRecorder, duration,
                target, queueToken)
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
    if !queueStartTime.IsZero() {
        duration := time.Since(queueStartTime).Seconds()
        pickQueueDurationMetric.Record(metricsRecorder, duration,
            target, queueToken)
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

### Metric Instrument Registration

#### Go

```go
import estats "google.golang.org/grpc/experimental/stats"

var pickQueueDurationMetric = estats.RegisterFloat64Histo(
    estats.MetricDescriptor{
        Name:           "grpc.lb.pick_queue_duration",
        Description:    "EXPERIMENTAL. Time an RPC spent queued waiting " +
                        "for a load balancing pick.",
        Unit:           "s",
        Labels:         []string{"grpc.target"},
        OptionalLabels: []string{"grpc.lb.queue_reason"},
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
        List.of("grpc.target"),                // required labels
        List.of("grpc.lb.queue_reason"),       // optional labels
        false);                                // default disabled
```

#### C++ (Core)

```cpp
const auto kPickQueueDurationHandle =
    GlobalInstrumentsRegistry::RegisterDoubleHistogram(
        "grpc.lb.pick_queue_duration",
        "EXPERIMENTAL. Time an RPC spent queued waiting "
        "for a load balancing pick.",
        "s",
        /*label_keys=*/{"grpc.target"},
        /*optional_label_keys=*/{"grpc.lb.queue_reason"},
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

The queue reason token is exposed via the Picker rather than via `PickResult` or
the error return, because the existing queueing mechanisms differ across
languages. In Go, queueing is signaled by returning
`balancer.ErrNoSubConnAvailable` and the `PickResult` is discarded. In C++,
`PickResult::Queue` is an empty struct. In Java, `PickResult.withNoResult()`
returns a singleton. An optional interface or default-returning method on the
Picker aligns with each language's existing mechanism without requiring changes
to the error type.

The tracing enhancement reuses the same bounded queue reason token as the metric
label rather than introducing a separate high-cardinality trace reason string.
This keeps the API surface minimal — one token serves both metrics and traces —
and avoids the need for pickers to maintain two separate strings. If richer,
dynamic trace context (e.g., IP addresses, RLS keys) is needed in the future,
it can be added as additional span event attributes without changing the token
API.

`wrr_locality` and `outlier_detection` are treated as transparent because
neither makes independent queueing decisions. `wrr_locality` is a configuration
wrapper; `outlier_detection` manipulates subchannel state but the actual
queueing is decided by the child policy's picker. Adding prefixes for these
would increase token length without diagnostic value.

`grpc.method` is excluded as a label because LB queue duration is a property of
the channel's LB policy state, which is typically the same for all methods.
Including it would significantly increase cardinality without commensurate
diagnostic value. Per-method queue analysis can be correlated with existing
per-call attempt metrics.

## Implementation

Implementation will proceed in Go, Java, and C++ (Core), in that order.

[A66]: A66-otel-stats.md
[A72]: A72-open-telemetry-tracing.md
[A78]: A78-grpc-metrics-wrr-pf-xds.md
[A79]: A79-non-per-call-metrics-architecture.md
[A91]: A91-outlier-detection-metrics.md
[A94]: A94-subchannel-otel-metrics.md
[A56]: A56-priority-lb-policy.md
[A50]: A50-xds-outlier-detection.md