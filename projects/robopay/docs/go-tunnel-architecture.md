# Go Tunnel Architecture

## Overview

The Go tunnel is the payment gateway between the Fabric relay and the Zenoh robot transport. It enforces x402 payment verification, manages settlement, and provides discovery/status endpoints.

---

## Endpoints

| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| GET | `/robot` | none | Robot discovery — returns connected robots |
| GET | `/skills` | none | Skill discovery — returns available skills and prices |
| POST | `/action` | x402 | Execute a payment-validated action |
| GET | `/action/:action_id/status` | none | Check action status (pending/succeeded/failed) |

---

## POST /action Flow

```
Request arrives
    ↓
x402 middleware
    ├── No payment → 402 { payment_requirements }
    ├── Invalid authorization → 400 { error: "invalid_exact_evm_signature" }
    ├── Replay → 400 { error: "DUPLICATE_ACTION" }
    └── Valid → continue
    ↓
Parse action request
    ├── Missing identity fields → 400
    └── Valid → continue
    ↓
Publish to Zenoh: robot/tunnel/action
    ├── Publish fails → 502, no waiter created
    └── Publish succeeds → 202 { action_id, status_url }
    ↓
Background watcher
    ├── Subscribes to robot/tunnel/result
    ├── Correlates by action_id
    ├── On success → invokes settlement callback
    └── On failure/timeout → no settlement
```

---

## Settlement Gate

The settlement gate replaces the stock x402 middleware's immediate settlement:

```go
// Stock x402: settles as soon as handler returns < 400
// This tunnel: settles only when correlated result reports success

func (g *SettlementGate) HandleAction(c *gin.Context) {
    // 1. Verify payment (x402 half)
    // 2. Publish to Zenoh
    // 3. Return 202 with action_id
    // 4. Register settlement callback with watcher
}

func (g *SettlementGate) OnResult(actionID string, result Result) {
    if result.Status == "succeeded" {
        g.settle(actionID)  // invoke facilitator
    }
    // else: no settlement, authorization stays signed and unspent
}
```

---

## Idempotency

```go
type IdempotencyStore struct {
    mu       sync.RWMutex
    entries  map[string]*Entry  // key: robot_id:skill_id:idempotency_key
    filePath string             // persisted to disk
}

type Entry struct {
    ActionID        string
    Params          map[string]interface{}
    PaymentFingerprint string
    Outcome         string  // "pending" | "succeeded" | "failed"
    Timestamp       time.Time
}
```

| Scenario | Result |
| --- | --- |
| Same key, same request | `duplicate` — recorded outcome replayed |
| Same key, different params | `IDEMPOTENCY_PARAMS_CONFLICT` |
| Same key, different payment | `IDEMPOTENCY_PAYMENT_CONFLICT` |
| Same key after restart | Still one actuation (persisted to disk) |

---

## Tests

8 tests in `handlers_test.go` cover the settlement gate without a wallet or chain:

| Test | What it proves |
| --- | --- |
| `TestUnpaidActionReturns402` | Unpaid requests never reach Zenoh |
| `TestForgedAuthorizationReturns400` | Forged payments never reach Zenoh |
| `TestSuccessfulActionSettlesExactlyOnce` | Success → one settlement |
| `TestFailedActionNeverSettles` | Failure → no settlement |
| `TestSilentRobotNeverSettles` | Timeout → no settlement |
| `TestReplayRejected` | Duplicate action_id → 400 |
| `TestMissingIdentityFieldRefused` | Incomplete request → 400 before Zenoh |
| `TestFailedPublishNoWaiterLeft` | 502 → no waiter, no settlement |
