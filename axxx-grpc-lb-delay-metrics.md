# 📐 Comprehensive Design (gRFC): LB Delay Observability

*   **Author**: Antigravity & Madhav Bissa
*   **Status**: Draft (Approved)
*   **Created**: 2026-04-01

---

## 📖 Abstract

This document defines a cross-language design for tracking queuing delays in the gRPC Load Balancing (LB) policy tree. It generalizes concepts so they apply symmetrically to `grpc-go`, `grpc-java`, and C++ (`grpc-core`).

When an RPC is queued, the Load Balancing policy will return a **Policy Token** tracking the reason for the delay. The gRPC Channel will use this token to record precise temporal metrics and tracing events.

---

## 🏗️ Component 1: Core API (`LoadBalancer` / `Picker`)

We modify the `PickResult` object to include a `Token` string. Returns `Subchannel = null` AND `err == null` indicates a **Queue Action** if `Token` is set.

### 💻 Generic Pseudo-code:
```java
class PickResult {
    Subchannel subchannel; 
    CompletionCallback onDone;
    Metadata metadata;
    String token; // 👈 NEW
}

interface Picker {
    PickResult pick(PickArgs args);
}
```

---

## 🌉 Component 2: Channel Interceptor (`PickerWrapper`)

The Channel-side interceptor (which calls the Picker) tracks state transitions and measures time elapsed.

### 💻 Generic Pseudo-code:
```java
class PickerWrapper {
    String lastToken = "";
    Timestamp lastTokenTime;

    PickResult pick(RPC rpc) {
        while (true) {
            Picker currentPicker = getCurrentPicker();
            PickResult result = currentPicker.pick(rpc);

            if (result.subchannel == null && result.token != null) {
                
                incrementActiveWaiters(result.token); 
                recordTokenChange(result.token);
                syncWaitOnPickerUpdate();
                decrementActiveWaiters(result.token);
                continue;
            }

            if (result.subchannel != null && result.subchannel.isReady()) {
                recordTokenChange(""); 
                return result;
            }
        }
    }
}
```

---

## 🕵️‍♂️ Component 3: Policy Behavior (Standard Policies)

### a. PickFirst
Returns `"pf:connecting"` or `"pf:idle"`.

### b. RingHash
Returns `"ring_hash:building"` or `"ring_hash:connecting"`.

---

## 🔀 Component 4: Composite Policies (Failover / Aggregation)

Parent policies concatenate tokens.

### a. Priority Policy
Returns `"p0:pf:connecting"`.

---

## 🛰️ Component 5: Resolver-coupled & Intermediate Policies

### a. RLS (Route Lookup Service) Policy
Returns `"rls"` on miss/pending lookup.

### b. CDS (Cluster Discovery Service)
Returns `"cds:discovery"`.

### c. xDS Cluster Manager
Returns `"cluster_manager:cluster_a:rls"`.

---

## 🌍 Scheme-Independence (DNS vs xDS)

Design is agnostic of the resolution scheme. `PickerWrapper` intercepts the root picker.

---

## 🚨 Real-Time Tracking for Stuck RPCs (UpDownCounter)

Histograms only record data on **completion**. We introduce a real-time counter to solve this.

### 📐 Metric: `grpc.client.lb.active_queued_calls`
*   **Type**: UpDownCounter (Sum)
*   **Unit**: `{call}`
*   **Description**: Number of RPCs currently waiting for an LB policy to find a READY subchannel.

---

## 📊 Unified Metric Representation for All Delays

We establish a unified representation for **any type of delay** in gRPC.

### 📐 Metric Characteristics:
1.  **Instrument Type**: Histogram (to capture distribution, percentiles, and p99).
2.  **Standard Unit**: Fractional seconds (`s`).

### 📚 Domain Examples:
*   `grpc.client.lb.queue_duration`
*   `grpc.client.resolution_duration`

---

## 📈 Metric Population Mechanics (Sequence Examples)

This section details how tokens received by the channel translate into exact metric values and labels.

### 🏁 Example 1: The RLS-to-PickFirst Transition (Successful Pick)

Consider an RPC that wait for RLS lookup, then waits for a backend connection, then succeeds.

| Timeline | Action | `PickerWrapper` State Change | Metric Events |
| :--- | :--- | :--- | :--- |
| **T0 (0s)** | RPC starts. Picker returns `null, token="rls"` | `lastToken="rls"`, `lastTokenTime=T0` | ➕ `active_queued_calls{token="rls"}` = +1 |
| **T2 (2s)** | RLS succeeds! New Picker returns `null, token="p0:pf:connecting"` | Token switches `"rls"` ➜ `"p0:pf:connecting"`. `lastTokenTime=T2` | ➖ `active_queued_calls{token="rls"}` = -1<br>➕ `active_queued_calls{token="p0:pf:connecting"}` = +1<br>📊 **Emit** `lb_queue_duration{token="rls"}` = `2s` |
| **T3 (3s)** | TCP connection ready! New picker returns `Subchannel != null` | Token switches `"p0:pf:connecting"` ➜ `""`. Clear state. | ➖ `active_queued_calls{token="p0:pf:connecting"}` = -1<br>📊 **Emit** `lb_queue_duration{token="p0:pf:connecting"}` = `1s` |

*   **Total Observable Delay**: 3s (2s RLS + 1s PickFirst).
*   **Active Counter Check**: The sum of counter increments (+1, -1, +1, -1) equals **0**, ensuring no leakage.

---

### ⏳ Example 2: The Timeout Scenario (Indefinite Stall)

Consider an RPC that starts, gets stuck in RLS wait, and times out before RLS responds.

| Timeline | Action | `PickerWrapper` State Change | Metric Events |
| :--- | :--- | :--- | :--- |
| **T0 (0s)** | RPC starts. Picker returns `null, token="rls"` | `lastToken="rls"`, `lastTokenTime=T0` | ➕ `active_queued_calls{token="rls"}` = +1 |
| **T5 (5s)** | Context Context Deadline Exceeded (RPC fails) | Process exits blocking loop. Clear state. | ➖ `active_queued_calls{token="rls"}` = -1<br>📊 **Emit** `lb_queue_duration{token="rls"}` = `5s` |

*   **Outcome**: Even though the RPC failed and never got a ready subchannel, the system still records that it spent 5 seconds queueing due to `"rls"`. This ensures failure-inducing delays are visible!

---

## 📉 Tracing Visual Timeline

Span Events on token/state changes:
-   **Event Name**: `grpc.delay.event`
-   **Attributes**: `delay.type: "lb_queue"`, `previous_token: "rls"`, `new_token: "p0:pf:connecting"`.
