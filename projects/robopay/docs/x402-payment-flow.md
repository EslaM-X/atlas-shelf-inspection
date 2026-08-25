# x402 Payment Flow

## Overview

RoboPay uses the [x402](https://x402.org) protocol for payment-gated robot control. Every action requires a valid EIP-3009 authorization before the robot moves.

---

## Flow

```
1. Client discovers robot and price
   GET /robots/{id}/skills → [{ skill_id, price_usdc, asset, network }]

2. Client requests action without payment
   POST /action → 402 { payment_requirements: { amount, asset, network, payee } }

3. Client signs EIP-3009 authorization
   nonce = keccak256(action_id)
   signTransfer(payer, payee, amount, nonce, ...)

4. Client resubmits with authorization
   POST /action { payment: { authorization, txHash }, ... } → 202 { action_id, status_url }

5. Robot executes
   Zenoh: robot/tunnel/action → bridge → simulator → robot/tunnel/result

6. Settlement (only on success)
   POST /settle { action_id, result } → facilitator verifies → USDC transfer on-chain
```

---

## Security Properties

| Property | Mechanism |
| --- | ---|
| Unpaid actions never execute | x402 gate returns 402, action never published to Zenoh |
| Forged authorizations refused | Live facilitator verifies signature, returns `invalid_exact_evm_signature` |
| Failed executions never settle | Settlement gate checks result status before invoking facilitator |
| No double settlement | Idempotency store keyed on `robot_id + skill_id + idempotency_key` |
| Replay rejected | Same authorization nonce cannot be used twice; USDC tracks spent nonces |
| Amount bound to action | Authorization nonce = `keccak256(action_id)`, so each authorization is unique |

---

## Settlement Invariants

```python
# Invariant 1: Failed execution → no settlement
assert test_valid_payment_failure_no_settlement()  # PASS

# Invariant 2: Silent robot → no settlement
assert test_silent_robot_no_settlement()  # PASS

# Invariant 3: Exception mid-episode → no settlement
assert test_execution_exception_no_settlement()  # PASS

# Invariant 4: Replay → no settlement
assert test_replay_rejected()  # PASS

# Invariant 5: Success → settle exactly once
assert test_settle_on_success_only()  # PASS
```

---

## On-Chain Verification

Anyone can verify a settlement without trusting the artifact:

```python
from web3 import Web3

# Recompute the authorization nonce from the action_id
nonce = Web3.keccak(text="atlas-inspect-1787242629")

# Query the USDC contract for AuthorizationUsed event
# If nonce appears in the event, the authorization was spent
# If not, the action was never paid for
```

The payer holds no ETH under EIP-3009 — the signer authorizes and the facilitator pays the gas. This means the payer's wallet needs only the signing key, not ETH for gas.
