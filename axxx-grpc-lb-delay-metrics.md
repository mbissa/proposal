# gRFC: Load Balancer Delay Observability

**Author**: Madhav Bissa  
**Status**: Draft  
**Created**: 2026-04-16

---

## 1. Abstract and Objective
This document proposes a cross-language architectural design for tracking and exposing queuing delays within the gRPC Load Balancing (LB) policy tree. Currently, when an RPC is queued waiting for a `READY` subchannel, the client channel is blind to which specific layer of the LB tree is causing the delay.

The objective is to programmatically identify the exact cause of RPC latency at the LB component level without degrading the high-performance hot path of gRPC or causing unbounded metric cardinality.

## 2. Scope and Constraints
* **Supported Languages**: `grpc-go`, `grpc-java`, and C++ (`grpc-core`).
* **Agnosticism**: Must function independently of the resolution scheme (DNS, xDS, RLS).
* **Performance Constraint**: Zero dynamic memory allocations on the `Picker.Pick()` hot path.
* **Compatibility Constraint**: Must remain strictly non-breaking for existing custom load balancers.

## 3. High-Level Architecture (The Bifurcated Schema)
The solution transitions the system from "blind blocking" to "state-aware blocking" using a bifurcated schema to satisfy both strict metrics and highly-contextual tracing.

1.  **Queue Tokens & Reasons**: LB policies pre-compute a strict, bounded `MetricToken` AND an optional, dynamic, human-readable `TraceReason` when their `Picker` is constructed.
2.  **State Exposure**: When `Pick()` determines an RPC must queue, it returns references to both strings alongside `ErrNoSubConnAvailable`.
3.  **The Interceptor**: The Channel-side wrapper records the strict token for metrics, and the dynamic reason for trace span annotations.

> **Rationale (Bifurcation):** Distributed tracing can handle unique, dynamic strings (e.g., `"RLS: child 10.0.0.1 attempting to connect"`). Metrics systems (Prometheus) will crash (OOM) if dynamic strings are used as labels due to cardinality explosion. The API must accept both a strict enum-like token for metrics and a dynamic string for tracing to safely satisfy all observability needs.

## 4. API Changes (Cross-Language)

### 4.1 gRPC-Go
```go
// In package balancer
type PickResult struct {
    SubConn SubConn
    Done    func(DoneInfo)
    Metadata metadata.MD
    
    // MetricToken is the strict, bounded reason an RPC is queued (e.g., "rls:lookup_pending").
    MetricToken string 
    // TraceReason is an optional, dynamic string for span annotations.
    TraceReason string 
}
```

### 4.2 gRPC-Java
```java
public abstract static class PickResult {
    // Default methods ensure non-breaking backwards compatibility
    public String getMetricToken() { return ""; }
    public String getTraceReason() { return ""; }
    
    public static PickResult withQueue(String metricToken, String traceReason) { /* Impl */ }
}
```

### 4.3 gRPC-Core (C++)
The internal `PickResult::Queue` struct will be extended from a single string to a dual-field structure:
```cpp
struct Queue {
    absl::string_view metric_token;
    std::string trace_reason;

    explicit Queue(absl::string_view token, std::string reason = "") 
        : metric_token(token), trace_reason(std::move(reason)) {}
};
```

## 5. Observability Schema (Metrics & Labels)

### 5.1 Metrics Definition
| Metric Name | Instrument Type | Unit | Description |
| :--- | :--- | :--- | :--- |
| `grpc.client.lb.queue_duration` | Histogram | Seconds (s) | Measures the time spent waiting under a specific LB delay token. Recorded upon transition to a new token or completion of the wait. |
| `grpc.client.lb.active_queued_calls` | UpDownCounter | {call} | The real-time count of RPCs currently blocked on a specific token. |

### 5.2 Label (Attribute) Definitions
| Label Name | Type | Description / Example |
| :--- | :--- | :--- |
| `grpc.target` | String | The target URI the channel was created with (e.g., `xds:///my-service`). |
| `grpc.method` | String | The fully qualified RPC method name (e.g., `/helloworld.Greeter/SayHello`). |
| `grpc.lb.delay_reason` | String | The strict `MetricToken` constructed by the LB policy tree (e.g., `cluster_manager:priority_p0:pick_first:connecting`). |

## 6. Exhaustive Metric Token Semantics (Valid Values)
Dynamic values are **strictly forbidden** in `MetricToken`s. Tokens use a standardized grammar: `[Delegating Prefix:]...[Terminal State]`.

### 6.1 Terminal State Tokens (Leaf Nodes & Overloads)
| Policy / Scenario | Terminal Token Value |
| :--- | :--- |
| `pick_first` | `pick_first:connecting` |
| `round_robin` | `round_robin:connecting` |
| `ring_hash` | `ring_hash:building_ring` |
| `ring_hash` | `ring_hash:connecting` |
| `ring_hash` | `ring_hash:fallback_scanning` |
| `rls` | `rls:lookup_pending` |
| `rls` | `rls:throttled_backoff` |
| `outlier_detection` | `outlier_detection:quarantine_exhausted` |
| `transport` (C-Core) | `transport:max_concurrent_streams` |

### 6.2 Delegating Prefix Tokens (Parent Nodes)
| Policy | Prefix Value | Rationale |
| :--- | :--- | :--- |
| `priority` | `priority_p{index}:` | Exposes the 10s `initTimer` failover progression. |
| `cluster_manager` | `cluster_manager:` | Dynamic cluster names are intentionally omitted for cardinality safety. |
| `rls` | `rls:` | Prepended when a lookup succeeds and routes to a child policy. |
| `cds` | `cds:discovery_pending` or `cds:` | Emits pending while waiting for xDS, otherwise prepends. |
| `wrr_locality` | *(Transparent)* | Transparent config transformer; passes child tokens upward unmodified to reduce string bloat. |

## 7. Architectural Boundaries & Low-Level Mechanics
A critical nuance of this design is establishing the exact boundary between the "Channel" (gRPC framework) and the "LB Policy".

> **Rationale (Telemetry Ownership):** The LB Policy **must not** emit these metrics. Queueing decisions are per-RPC. An LB Policy knows its internal state is "Connecting", but it does not know if 1 or 1,000 RPCs are waiting on it. The Channel Wrapper physically buffers the RPCs, making it the only component capable of accurately tracking time and count. The Channel Wrapper extracts the `estats.MetricsRecorder` initialized by the parent Channel.

### 7.1 Language-Specific Nuances
The queue duration and state-tracking logic must be injected into the specific wrapper for each language, acknowledging their unique architectural paradigms:
* **Go (`pickerWrapper.go`):** Uses a blocking `for {}` loop and a Go channel (`pw.blockingCh`). The metrics logic is injected directly into this synchronous loop, measuring time between select wakeups. Context cancellations must strictly decrement the UpDownCounter via a `defer` block.
* **Java (`DelayedClientTransport.java`):** Uses an event-driven queue. When an RPC queues, it is wrapped in a `DelayedClientCall`. Metric extraction and token-transition logic happen in the re-processing loop when the LB provides a new `SubchannelPicker` and the framework iterates over the pending RPC queue.
* **C-Core (`client_channel_filter.cc`):** Uses an interceptor-like filter stack. Queued picks are added to a linked list inside `ClientChannelFilter::LoadBalancedCall`. The filter walks this list and synchronously re-attempts picks when a new Picker arrives, utilizing the internal `ExecCtx` time for metric calculation.

## 8. Tracing Schema (Structured Context)
To provide visual timelines inside Jaeger/Zipkin, the `PickerWrapper` emits Span Events on the active RPC span on state transitions.

> **Rationale (Why include both?):** The strict `MetricToken` is highly valuable for programmatic filtering and ensuring consistency between metrics dashboards and traces. However, tracing is fundamentally designed for Root-Cause Analysis (RCA). By including the dynamic `TraceReason`, operators instantly gain the rich, contextual data (e.g., specific failing IPs) needed to solve an outage without digging through application logs.

The trace event must emit **both** the strict token and the dynamic reason to achieve the ultimate observability posture:
* **Event Name**: `grpc.lb.delay_event`
* **Attributes**:
    * `grpc.lb.metric_token`: The strict, low-cardinality token (e.g., `"rls:cds:pick_first:connecting"`).
    * `grpc.lb.dynamic_reason`: The high-cardinality, contextual string (e.g., `"RLS resolved key 'user-123'; connecting to 10.4.5.22 failed (Connection Refused)"`).

## 9. Resolved Architectural Decisions

### Resolved Decision: Tracing Span Event Debouncing
**Context**: A flapping resolver can cause a policy to switch between `Connecting` and `TransientFailure` hundreds of times per second. Unchecked Span Event emissions will cause OOM crashes in telemetry agents.

**Resolution: State-Change Debouncing.** We will NOT use a time-based (e.g., 10ms) debounce window. Instead, the Channel Wrapper will maintain a state variable (`last_metric_token` and `last_trace_reason`). Metrics and Trace annotations are **only emitted when the string transition differs from the previous state**. If a policy remains stuck in `"pick_first:connecting"` across 10 separate Picker updates, only the initial transition is recorded.

## 10. Open Architectural Decisions (For Review)

### Decision 1: Synchronous Pick Stall Tracking
**Context**: Policies like `RingHash` or `RLS` might block the `Pick()` execution synchronously (CPU delay) to rebuild data structures.
* **Option A: Emit as a Tracing Event Only**. Wrap the `p.Pick(info)` call in a timer. If > 5ms, emit `grpc.lb.sync_pick_stall`.
* **Option B: Introduce `grpc.client.lb.sync_pick_duration` Metric**. Create a new Histogram metric.

### Decision 2: UpDownCounter Cleanup Mechanism (Go-Specific)
**Context**: Rapid destruction of sub-balancers can occur while RPCs are actively queued. The active count must be reliably decremented to prevent metric leaks.
* **Option A: Strict Defer Pattern**. Use a `defer` block to always evaluate and decrement `lastToken` upon exiting the wait loop.
* **Option B: Explicit Cleanup in Select Blocks**. Manually decrement the counter inside the `ctx.Done()` block.

---

## 11. Exhaustive Configuration Permutations & Combinations
This section lists the configuration permutations, routing combinations, and state triggers that dictate exactly which metrics (`grpc.target`, `grpc.method`, `grpc.lb.delay_reason`) and trace span events will be emitted.

### Category I: Target URI & Service Config Changes (Impacts `grpc.target` & `grpc.method`)
1. **Config:** Target set to `dns:///backend.svc.cluster.local:80` | **Trigger:** DNS resolution delay | **Label:** `grpc.target="dns:///backend.svc.cluster.local:80"`, `delay_reason="pick_first:connecting"`
2. **Config:** Target set to `xds:///payments.global.api` | **Trigger:** Waiting for xDS CDS | **Label:** `grpc.target="xds:///payments.global.api"`, `delay_reason="cds:discovery_pending"`
3. **Config:** Target set to `rls:///dynamic-router` | **Trigger:** RLS lookup | **Label:** `grpc.target="rls:///dynamic-router"`, `delay_reason="rls:lookup_pending"`
4. **Config:** Target set to `ipv4:10.0.0.1:8080` (Direct) | **Trigger:** TCP handshake delay | **Label:** `grpc.target="ipv4:10.0.0.1:8080"`, `delay_reason="pick_first:connecting"`
5. **Config:** Service Config defines default route | **Trigger:** RPC to `/User/Get` | **Label:** `grpc.method="/User/Get"`
6. **Config:** Service Config adds Method Config for `/User/Update` | **Trigger:** RPC to `/User/Update` | **Label:** `grpc.method="/User/Update"`
7. **Config:** Service Config fallback route matched | **Trigger:** Unknown RPC `/User/Unknown` | **Label:** `grpc.method="/User/Unknown"`
8. **Config:** Target URI changes from `dns:///` to `xds:///` via app restart | **Trigger:** xDS setup | **Label:** `grpc.target` switches to `xds:///...`
9. **Config:** Service Config applies different LB policies per method (Method A uses PickFirst, Method B uses RoundRobin) | **Trigger:** Call Method A | **Label:** `delay_reason="pick_first:connecting"`
10. **Config:** Service Config per-method LB (Method B uses RoundRobin) | **Trigger:** Call Method B | **Label:** `delay_reason="round_robin:connecting"`

### Category II: Leaf Policy Configurations (PickFirst, RoundRobin, RingHash)
11. **Config:** `[pick_first]` | **Trigger:** Channel enters IDLE, RPC wakes it up | **Label:** `pick_first:connecting` | **Span:** `previous="", new="pick_first:connecting"`
12. **Config:** `[pick_first]` | **Trigger:** TCP connection drops, enters TRANSIENT_FAILURE (TF) | **Label:** `pick_first:connecting` (reconnect attempt)
13. **Config:** `[round_robin]` | **Trigger:** Subchannel list empty pending DNS | **Label:** `round_robin:connecting`
14. **Config:** `[round_robin]` | **Trigger:** All RR subchannels drop to TF | **Label:** `round_robin:connecting`
15. **Config:** `[round_robin]` | **Trigger:** DNS returns 5 IPs, 1 connects, 4 fail | **Metric:** No queueing emitted (RR is READY if at least 1 is connected).
16. **Config:** `[ring_hash]` (Default config) | **Trigger:** Waiting for DNS to build ring | **Label:** `ring_hash:building_ring`
17. **Config:** `[ring_hash]` (`min_ring_size` increased to 100,000) | **Trigger:** Massive CPU stall during rebuild | **Trace:** Synchronous pick stall event emitted (Metrics unaffected).
18. **Config:** `[ring_hash]` | **Trigger:** Primary hashed endpoint is in CONNECTING | **Label:** `ring_hash:connecting`
19. **Config:** `[ring_hash]` | **Trigger:** Primary endpoint TF, looking ahead to alternative 1 | **Label:** `ring_hash:fallback_scanning`
20. **Config:** `[ring_hash]` (`max_ring_size` reduced) | **Trigger:** Rebuilding smaller ring | **Label:** `ring_hash:building_ring`
21. **Config:** `[ring_hash]` | **Trigger:** Primary fails, fallback 1 fails, fallback 2 CONNECTING | **Label:** `ring_hash:fallback_scanning`
22. **Config:** `[ring_hash]` | **Trigger:** Endpoint weight changes in config | **Label:** `ring_hash:building_ring` (Ring recalculation)
23. **Config:** `[ring_hash]` | **Trigger:** Request lacks Hash attribute | **Label:** `ring_hash:connecting` (Defaults to first address hash)
24. **Config:** `[pick_first]` | **Trigger:** Connection successful | **Span:** `previous="pick_first:connecting", new=""`
25. **Config:** `[pick_first]` | **Trigger:** Context timeout while queueing | **Metric:** `active_queued_calls` is decremented.

### Category III: Outlier Detection Config Changes
26. **Config:** `[outlier_detection]` wrapping `[pick_first]` | **Trigger:** Initial connection | **Label:** `pick_first:connecting` (OD is transparent during normal ops).
27. **Config:** OD `success_rate_ejection` enabled | **Trigger:** Target hits 5xx limits, ejected | **Label:** `outlier_detection:quarantine_exhausted` (If leaf fails to pick).
28. **Config:** OD `failure_percentage_ejection` enabled | **Trigger:** 100% of nodes ejected, RR requests subchannel | **Label:** `outlier_detection:quarantine_exhausted`
29. **Config:** OD `max_ejection_percent` set to 50% | **Trigger:** 50% nodes ejected, remaining 50% enter TF | **Label:** `round_robin:connecting` (Limit reached, OD delegates failure to leaf).
30. **Config:** OD `max_ejection_time` expires | **Trigger:** Nodes unejected, DNS reconnects | **Label:** `round_robin:connecting`
31. **Config:** `[outlier_detection]` wrapping `[ring_hash]` | **Trigger:** Hashed node ejected | **Label:** `ring_hash:fallback_scanning` (RingHash scans past the OD-ejected node).
32. **Config:** OD `enforcing_success_rate` set to 0% (Disabled) | **Trigger:** Nodes fail but aren't ejected | **Label:** `round_robin:connecting` (OD quarantine metric never fires).
33. **Config:** OD `enforcing_failure_percentage` set to 100% | **Trigger:** All nodes fail | **Label:** `outlier_detection:quarantine_exhausted`
34. **Config:** OD wraps Priority | **Trigger:** P0 nodes ejected | **Label:** `priority_p1:pick_first:connecting` (Priority fails over successfully due to OD ejection).
35. **Config:** OD wraps Priority | **Trigger:** ALL nodes in ALL priorities ejected | **Label:** `outlier_detection:quarantine_exhausted`

### Category IV: Priority Policy Configs (Failovers & Timers)
36. **Config:** 2-Level Priority (`P0`, `P1`) -> PickFirst | **Trigger:** Initial start | **Label:** `priority_p0:pick_first:connecting`
37. **Config:** Priority | **Trigger:** P0 fails `initTimer` (10s limit) | **Label:** `priority_p1:pick_first:connecting`
38. **Config:** Priority | **Trigger:** P0 timer expires | **Span:** `previous="priority_p0:pick_first:connecting", new="priority_p1:pick_first:connecting"`
39. **Config:** Priority | **Trigger:** P0 connects after 12s | **Metric:** Emits `queue_duration` of 10s for P0 (timeout), and 2s for P1. 
40. **Config:** 3-Level Priority (`P0`, `P1`, `P2`) | **Trigger:** P0 fails, P1 fails | **Label:** `priority_p2:pick_first:connecting`
41. **Config:** Priority | **Trigger:** P0 recovers while P1 is CONNECTING | **Label:** `priority_p0:pick_first:connecting` (Aggressive fail-back).
42. **Config:** Priority | **Trigger:** P1 recovers while P2 is CONNECTING | **Label:** `priority_p1:pick_first:connecting`
43. **Config:** Priority wrapping `ring_hash` at P0 | **Trigger:** P0 ring building | **Label:** `priority_p0:ring_hash:building_ring`
44. **Config:** Priority wrapping `ring_hash` | **Trigger:** P0 ring hash scanning | **Label:** `priority_p0:ring_hash:fallback_scanning`
45. **Config:** Priority | **Trigger:** P1 `initTimer` expires, no P2 configured | **Label:** `priority_p1:pick_first:connecting` (Stays on lowest priority indefinitely).
46. **Config:** Priority | **Trigger:** P0 reports TF immediately (fast-fail) | **Label:** `priority_p1:pick_first:connecting` (Bypasses 10s timer).
47. **Config:** Priority | **Trigger:** P0 fast-fails, P1 fast-fails | **Label:** `priority_p1:pick_first:connecting` (Parks on lowest).
48. **Config:** Priority config updated via xDS to drop P1 | **Trigger:** Live config update | **Label:** `priority_p0:pick_first:connecting` (If parked on P1, forcefully moves to P0).
49. **Config:** Priority | **Trigger:** Flapping P0 (TF -> CONNECTING rapidly) | **Trace:** State-change debouncer suppresses duplicate span events.
50. **Config:** Priority wrapping WRR | **Trigger:** P0 connecting | **Label:** `priority_p0:round_robin:connecting` (WRR is transparently omitted).

### Category V: WRR Locality Config Changes
51. **Config:** `[wrr_locality]` -> `[round_robin]` | **Trigger:** Initial connection | **Label:** `round_robin:connecting` (Prefix `wrr_locality:` intentionally stripped for cardinality).
52. **Config:** `[wrr_locality]` | **Trigger:** Locality weights change from 10 to 50 | **Label:** `round_robin:connecting` (Subchannels reconnect).
53. **Config:** `[wrr_locality]` | **Trigger:** Config missing LocalityWeight | **Metric:** No queueing emitted (RPC fails immediately with config error).
54. **Config:** `[wrr_locality]` -> `[pick_first]` | **Trigger:** Node disconnects | **Label:** `pick_first:connecting`
55. **Config:** Priority -> WRR -> RoundRobin | **Trigger:** P0 WRR nodes drop | **Label:** `priority_p1:round_robin:connecting`

### Category VI: RLS (Route Lookup Service) Caching & Throttling
56. **Config:** `[rls]` enabled | **Trigger:** RPC starts, key not in cache | **Label:** `rls:lookup_pending`
57. **Config:** `[rls]` | **Trigger:** Control-plane RLS server responds | **Span:** `previous="rls:lookup_pending", new="rls:pick_first:connecting"`
58. **Config:** `[rls]` | **Trigger:** RLS server takes 2s to reply | **Metric:** `queue_duration` for `rls:lookup_pending` = 2.0s
59. **Config:** `[rls]` | **Trigger:** RLS server returns `RESOURCE_EXHAUSTED` | **Label:** `rls:throttled_backoff`
60. **Config:** `[rls]` | **Trigger:** RPC starts, key is in backoff state | **Label:** `rls:throttled_backoff`
61. **Config:** `[rls]` | **Trigger:** Backoff timer expires, retrying lookup | **Label:** `rls:lookup_pending`
62. **Config:** `[rls]` | **Trigger:** RLS server returns target, target is CONNECTING | **Label:** `rls:pick_first:connecting`
63. **Config:** `[rls]` | **Trigger:** RLS cache hit (valid target) | **Label:** `rls:pick_first:connecting` (If data-plane target is down).
64. **Config:** `[rls]` | **Trigger:** RLS cache hit, target is READY | **Metric:** No delay emitted (0s).
65. **Config:** `[rls]` | **Trigger:** RLS cache entry stale, background refresh triggered | **Metric:** No delay (Uses stale target while refreshing).
66. **Config:** `[rls]` | **Trigger:** Stale refresh fails, target still connecting | **Label:** `rls:pick_first:connecting`
67. **Config:** `[rls]` (`defaultTarget` configured) | **Trigger:** RLS lookup fails entirely | **Label:** `rls:pick_first:connecting` (Routes to default target).
68. **Config:** `[rls]` (no `defaultTarget`) | **Trigger:** RLS lookup fails entirely | **Metric:** RPC fails with Unavailable, no queue duration emitted.
69. **Config:** `[rls]` | **Trigger:** Target drops to TF, RLS suppresses state | **Label:** `rls:pick_first:connecting` (Picker evaluates frozen state).
70. **Config:** `[rls]` | **Trigger:** Cache evicted due to LRU size limit | **Label:** `rls:lookup_pending` (Next RPC triggers miss).
71. **Config:** `[rls]` -> `[round_robin]` | **Trigger:** RLS success, RR connecting | **Label:** `rls:round_robin:connecting`
72. **Config:** `[rls]` -> `[ring_hash]` | **Trigger:** RLS success, building ring | **Label:** `rls:ring_hash:building_ring`

### Category VII: xDS Control Plane (ClusterManager & CDS) Configs
73. **Config:** xDS enabled | **Trigger:** Waiting for CDS response from xDS server | **Label:** `cds:discovery_pending`
74. **Config:** xDS CDS returns | **Trigger:** ClusterManager creates child | **Span:** `previous="cds:discovery_pending", new="cluster_manager:pick_first:connecting"`
75. **Config:** `[cluster_manager]` -> `[pick_first]` | **Trigger:** Initial setup | **Label:** `cluster_manager:pick_first:connecting`
76. **Config:** `[cluster_manager]` -> `[round_robin]` | **Trigger:** RR connecting | **Label:** `cluster_manager:round_robin:connecting`
77. **Config:** `[cluster_manager]` -> `[ring_hash]` | **Trigger:** RingHash connecting | **Label:** `cluster_manager:ring_hash:connecting`
78. **Config:** `[cluster_manager]` | **Trigger:** RPC requests `cluster_A`, cluster missing from config | **Metric:** Unavailable error (no queueing).
79. **Config:** `[cluster_manager]` | **Trigger:** Config update removes `cluster_B` | **Metric:** `SubBalancerCloseTimeout=0` instantly kills picker; `active_queued_calls` for `cluster_B` decrements.
80. **Config:** `[cluster_manager]` -> `[priority]` (P0) -> `[pick_first]` | **Trigger:** P0 connects | **Label:** `cluster_manager:priority_p0:pick_first:connecting`
81. **Config:** `[cluster_manager]` -> `[priority]` | **Trigger:** P0 fails to P1 | **Label:** `cluster_manager:priority_p1:pick_first:connecting`
82. **Config:** `[cluster_manager]` | **Trigger:** Cluster transitions TF -> CONNECTING | **Label:** `cluster_manager:pick_first:connecting` (State flapping suppressed by CM, but PickerWrapper still tracks actual picker).
83. **Config:** xDS CDS | **Trigger:** xDS Server unreachable, CDS timeout | **Metric:** RPC fails with UNAVAILABLE (no LB queue delay).
84. **Config:** xDS | **Trigger:** Target URI changed to point to a new xDS server | **Label:** `cds:discovery_pending`
85. **Config:** `[cluster_manager]` -> `[outlier_detection]` -> `[round_robin]` | **Trigger:** All nodes ejected | **Label:** `cluster_manager:outlier_detection:quarantine_exhausted`
86. **Config:** `[cluster_manager]` -> `[wrr_locality]` -> `[round_robin]` | **Trigger:** Nodes connect | **Label:** `cluster_manager:round_robin:connecting` (WRR omitted).
87. **Config:** xDS | **Trigger:** Complex tree failover (P0 -> P1 -> TF) | **Trace:** Span event history shows exact token transitions tracking the failover cascade.

### Category VIII: RLS + xDS Integration (ClusterSpecifier Plugin)
88. **Config:** xDS with RLS ClusterSpecifier | **Trigger:** RLS Cache miss | **Label:** `rls:lookup_pending`
89. **Config:** xDS + RLS | **Trigger:** RLS Server returns a cluster name | **Span:** `previous="rls:lookup_pending", new="rls:cds:discovery_pending"`
90. **Config:** xDS + RLS | **Trigger:** Waiting for CDS to resolve the RLS-provided cluster | **Label:** `rls:cds:discovery_pending`
91. **Config:** xDS + RLS | **Trigger:** CDS resolves, Priority P0 connects | **Label:** `rls:cds:priority_p0:pick_first:connecting`
92. **Config:** xDS + RLS | **Trigger:** P0 fails, P1 connects | **Label:** `rls:cds:priority_p1:pick_first:connecting`
93. **Config:** xDS + RLS | **Trigger:** RLS Throttled | **Label:** `rls:throttled_backoff`
94. **Config:** xDS + RLS | **Trigger:** RLS Cache hit, but CDS cluster was removed from xDS server | **Label:** `rls:cds:discovery_pending` (Hangs waiting for resource).
95. **Config:** xDS + RLS | **Trigger:** CDS resolves to RingHash | **Label:** `rls:cds:ring_hash:building_ring`
96. **Config:** xDS + RLS | **Trigger:** RLS returns cluster, CDS resolves to RoundRobin, nodes drop | **Label:** `rls:cds:round_robin:connecting`

### Category IX: C-Core Specifics & HTTP/2 Transport State
97. **Config:** C-Core, `[pick_first]` | **Trigger:** HTTP/2 `MAX_CONCURRENT_STREAMS` limit hit | **Label:** `transport:max_concurrent_streams`
98. **Config:** C-Core, `[round_robin]` | **Trigger:** Subchannel is READY, but streams exhausted | **Label:** `transport:max_concurrent_streams`
99. **Config:** C-Core, Priority | **Trigger:** P0 READY, streams exhausted | **Label:** `transport:max_concurrent_streams` (Priority tree resolves successfully, but transport blocks).
100. **Config:** C-Core | **Trigger:** Stream frees up | **Span:** `previous="transport:max_concurrent_streams", new=""`
101. **Config:** C-Core, `[pick_first]` | **Trigger:** Connection drops, returns to CONNECTING | **Label:** `pick_first:connecting`
102. **Config:** Go/Java (Non-C-Core) | **Trigger:** `MAX_CONCURRENT_STREAMS` limit hit | **Metric:** No LB queue duration emitted (Queued invisibly in transport layer).

### Category X: Lifecycle Transitions & Debouncing Edge Cases
103. **Trigger:** RPC Context deadline exceeded while `pick_first:connecting` | **Metric:** `queue_duration` emitted (calculated up to timeout point). `active_queued_calls` decremented by 1.
104. **Trigger:** Application explicitly cancels Context while `priority_p0:round_robin:connecting` | **Metric:** `active_queued_calls` decremented by 1.
105. **Trigger:** Rapid Flapping (TF -> CONNECTING -> TF in 2ms) | **Trace:** Only the first state change emits a `grpc.lb.delay_event` span annotation (State-Change Debouncing).
106. **Trigger:** Rapid Flapping | **Metric:** `queue_duration` is NOT recorded for micro-flaps that do not change the metric token string.
107. **Trigger:** Subchannel READY | **Metric:** `queue_duration` recorded for the final token. `active_queued_calls` decremented by 1.
108. **Trigger:** ClientConn gracefully closed while 50 RPCs are queued | **Metric:** 50 strict `defer` decrements occur; `active_queued_calls` drops to exactly 0.
109. **Trigger:** Token string identical across 5 successive `UpdateState` pushes | **Metric/Trace:** Ignored by PickerWrapper; considered a continuous block.
110. **Trigger:** `ClientConn` metrics explicitly disabled via `DialOption` | **Metric:** No `queue_duration` or `active_queued_calls` recorded (Hooks dynamically bypass).
111. **Trigger:** RLS child policy map destroyed dynamically | **Metric:** Any RPC queued on `rls:cds:connecting` instantly decrements.
112. **Trigger:** Pick is completely synchronous and instant (Cache hit + READY connection) | **Metric:** No histogram emission (Zero queuing).
113. **Trigger:** Pick blocks synchronously on CPU for 10ms (Ring rebuild) | **Metric:** Missed by standard queue tracking (As per Open Decision 1 Option A).
114. **Trigger:** Dynamic `TraceReason` changes (e.g., Target IP updates in string) but `MetricToken` is unchanged | **Trace:** New Span Annotation emitted. **Metric:** Ignored (Token unchanged).
115. **Trigger:** `grpc.target` contains an IPv6 address string | **Label:** `grpc.target="ipv6:[::1]:8080"` emitted correctly.