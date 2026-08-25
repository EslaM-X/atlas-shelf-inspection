# Validation Report — Atlas Shelf Inspection

**Profile:** `boston-dynamics.atlas.mujoco-pybullet-webots-shelf-inspection.v1`
**PR:** [#120](https://github.com/fabricfoundation/RoboPay/pull/120)
**Author:** EslaM-X
**Date:** 2026-08-19

---

## Tier 1 Requirements

| Requirement | Status | Evidence |
| --- | --- | --- |
| Simulator-only | ✅ PASS | MuJoCo, PyBullet, Webots — no hardware in the repo |
| Approved simulator | ✅ PASS | MuJoCo 3.11 primary; Webots R2025a and PyBullet cross-check |
| Policy / controller triggered | ✅ PASS | State machine + DLS resolved-rate IK, re-solved each tick |
| Not a replayed trajectory | ✅ PASS | No recorded trajectory; test asserts different commands for different poses |
| Not a built-in demo motion | ✅ PASS | Controller and task written here; upstream contributes only the robot |
| Simulator state metrics | ✅ PASS | Per-target error, targets completed, pelvis height, collisions, duration |
| Sim-to-Sim validation | ✅ PASS | Same URDF/task/controller on 3 engines, 3.20 mm spread (limit 50 mm) |
| Payment-validated action → Zenoh → simulator | ✅ PASS | Verified action published on `robot/tunnel/action`, executed, answered |
| Real Go tunnel gates the action | ✅ PASS | Tunnel refuses unpaid (402) and forged (400); neither reaches simulator |
| Forged payment refused | ✅ PASS | Live facilitator returns `invalid_exact_evm_signature` |
| One actuation per payment | ✅ PASS | Durable idempotency store, surviving restart |
| Clean, easy-to-understand code | ✅ PASS | No vendored binaries, one source of truth per concept |

---

## Cross-Engine Results

| Metric | MuJoCo | PyBullet | Webots R2025a |
| --- | --- | --- | --- |
| Status | **success** | **success** | **success** |
| Targets reached | **3 / 3** | **3 / 3** | **3 / 3** |
| Mean EE error | **9.54 mm** | **12.18 mm** | **8.98 mm** |
| Max EE error | 13.49 mm | 19.79 mm | 12.33 mm |
| Min pelvis height | **0.9084 m** | **0.9395 m** | **0.8982 m** |
| Falls | **0** | **0** | **0** |
| Collisions | **0** | **0** | **0** |
| Duration | 4.61 s | 4.98 s | 7.70 s |

**Verdict:** PASS — 3.20 mm spread (limit 50 mm)

---

## Payment Verification

| Test | Result |
| --- | --- |
| Unpaid action → 402 | ✅ |
| Wrong amount → 400 | ✅ |
| Malformed txHash → 400 | ✅ |
| Valid receipt → 202, executed 3/3 | ✅ |
| Replayed receipt → 400 | ✅ |
| Forged authorization → refused by live facilitator | ✅ |
| Failed execution → never settled | ✅ |
| Successful execution → settled exactly once | ✅ |

---

## Settlement Evidence

| Field | Value |
| --- | --- |
| Action | `act-paid-de66513f791b` |
| Tx | `0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39` |
| Block | 45706216 |
| Amount | 0.001 USDC |
| Network | Base Sepolia |
| Binding | `keccak256(action_id)` = authorization nonce |

---

## Conclusion

All Tier 1 requirements met. Payment-gated robot execution with on-chain settlement, cross-engine validation, and adversarial testing. Ready for review.
