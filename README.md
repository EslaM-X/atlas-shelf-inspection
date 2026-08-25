<div align="center">

# Atlas Shelf Inspection

**Payment-gated, policy-driven shelf inspection on a free-standing Boston Dynamics Atlas v4, validated across MuJoCo, PyBullet, and Webots R2025a from one pinned robot description.**

[![PR #120](https://img.shields.io/badge/PR-%23120-fabricfoundation%2FRoboPay-blue)](https://github.com/fabricfoundation/RoboPay/pull/120)
[![Tier 1](https://img.shields.io/badge/Tier-1-complete-brightgreen)](https://github.com/fabricfoundation/RoboPay/pull/120)
[![MuJoCo](https://img.shields.io/badge/MuJoCo-3.11-success)](#results)
[![PyBullet](https://img.shields.io/badge/PyBullet-latest-success)](#results)
[![Webots](https://img.shields.io/badge/Webots-R2025a-success)](#results)
[![Settlement](https://img.shields.io/badge/USDC-0.001-settled-blue)](https://sepolia.basescan.org/tx/0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

---

</div>

## What This Is

A complete implementation of **Tier 1** on the [RoboPay](https://github.com/fabricfoundation/RoboPay) protocol: a Boston Dynamics Atlas v4 robot performs a 3-target shelf inspection, where every action is **payment-gated** — the client must prove payment before the robot moves, and settlement happens on-chain only after successful execution.

**One continuous pass, nothing edited together:**

| Step | Result |
|---|---|
| Skill discovery | `inspect_shelf`, `stop` |
| Price discovery | **0.001 USDC** — read from relay, not hardcoded |
| Unpaid action | **402** — never reaches Zenoh |
| Paid action | **202 accepted** — `action_id` + status URL |
| Execution | **3/3 targets** on MuJoCo |
| Settlement | **0.001 USDC** on Base Sepolia |

---

## Quick Start

```bash
# Install dependencies
pip install -r projects/robopay/bridge/boston_dynamics/atlas_bridge/requirements.txt

# Fetch the Atlas v4 model
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.download_atlas_model

# Run the 116 tests
python -m pytest projects/robopay/bridge/boston_dynamics/atlas_bridge/tests -q

# Execute the shelf inspection
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.runner

# Cross-engine validation
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.sim2sim
```

---

## Project Map

```
atlas-shelf-inspection/
│
├── README.md                          ← you are here
├── LICENSE                            ← MIT
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
│
└── projects/robopay/                  ← PR #120 source code
    │
    ├── bridge/                        ← Python bridge
    │   ├── boston_dynamics/
    │   │   └── atlas_bridge/
    │   │       ├── control_core.py    ← DLS resolved-rate controller
    │   │       ├── kinematics.py      ← Jacobian + gravity from URDF
    │   │       ├── task.py            ← 3-target shelf inspection
    │   │       ├── episode.py         ← Episode recording
    │   │       ├── runner.py          ← MuJoCo execution
    │   │       ├── pybullet_runner.py ← PyBullet execution
    │   │       ├── webots_env.py      ← Webots execution
    │   │       ├── sim2sim.py         ← Cross-engine comparison
    │   │       ├── mujoco_env.py      ← MuJoCo environment
    │   │       ├── pybullet_env.py    ← PyBullet environment
    │   │       ├── actuators.py       ← Actuator mapping
    │   │       ├── model.py           ← Model loading
    │   │       ├── bridge.py          ← Zenoh bridge
    │   │       ├── payment.py         ← x402 payment
    │   │       ├── facilitator.py     ← Facilitator verification
    │   │       ├── idempotency.py     ← Idempotency store
    │   │       ├── x402.py            ← x402 middleware
    │   │       ├── relay.py           ← Relay client
    │   │       ├── demo_*.py          ← 4 demo scripts
    │   │       ├── real_paid_run.py   ← Live payment execution
    │   │       ├── settlement_evidence.py
    │   │       ├── reach_envelope.py
    │   │       ├── evidence_recording.py
    │   │       ├── visual_evidence.py
    │   │       ├── download_atlas_model.py
    │   │       ├── tests/             ← 116 tests
    │   │       │   ├── test_facilitator.py
    │   │       │   ├── test_idempotency.py
    │   │       │   ├── test_x402_payment_safety.py
    │   │       │   ├── test_inspection_task.py
    │   │       │   ├── test_kinematics.py
    │   │       │   ├── test_model_integrity.py
    │   │       │   └── test_bridge_contract.py
    │   │       └── webots/            ← Webots controller
    │   └── common/zenoh_bridge/
    │
    ├── tunnel/                        ← Go tunnel
    │   ├── cmd/main.go                ← Tunnel entry point
    │   └── internal/handlers/
    │       ├── handlers.go            ← Settlement gate
    │       ├── handlers_test.go       ← 8 gate tests
    │       └── discovery.go           ← Robot/skill discovery
    │
    ├── docs/evidence/                 ← Evidence artifacts
    │   ├── atlas-paid-action.gif      ← Full demo animation
    │   ├── atlas-shelf-inspection.gif ← Task animation
    │   ├── real-paid-run.json         ← Live settlement proof
    │   ├── fabric-relay-e2e.json      ← Full relay path
    │   ├── sim2sim-validation.json    ← Cross-engine results
    │   ├── mujoco-inspection-episode.json
    │   ├── pybullet-inspection-episode.json
    │   ├── webots-inspection-episode.json
    │   ├── go-tunnel-e2e-evidence.json
    │   ├── tunnel-e2e-evidence.json
    │   ├── onchain-settlement.json
    │   └── ...
    │
    ├── registry/                      ← Robot profile
    │   └── vendors/boston-dynamics/atlas/
    │       └── ...shelf-inspection.v1/
    │           ├── skills.yaml
    │           ├── functions.yaml
    │           ├── payment-policy.yaml
    │           ├── robot.profile.yaml
    │           ├── skill-catalog.json
    │           ├── execution-mapping.yaml
    │           └── tests/
    │
    ├── .github/workflows/
    │   └── boston-dynamics-atlas-tier1.yml  ← CI
    │
    └── NOTICE.md                      ← Licenses
```

---

## How It Works

```
Client                    Fabric Relay           Go Tunnel              Zenoh              Simulator
  │                           │                     │                    │                     │
  │  GET /robots/{id}/skills  │                     │                    │                     │
  │──────────────────────────>│                     │                    │                     │
  │  200 { skills, price }    │                     │                    │                     │
  │<──────────────────────────│                     │                    │                     │
  │                           │                     │                    │                     │
  │  POST /action (no payment)│                     │                    │                     │
  │──────────────────────────>│                     │                    │                     │
  │  402 { payment_requirements }                   │                    │                     │
  │<──────────────────────────│                     │                    │                     │
  │                           │                     │                    │                     │
  │  Sign EIP-3009 auth       │                     │                    │                     │
  │  nonce = keccak256(action_id)                   │                    │                     │
  │                           │                     │                    │                     │
  │  POST /action (with auth) │                     │                    │                     │
  │──────────────────────────>│                     │                    │                     │
  │                           │  WSS                │                    │                     │
  │                           │────────────────────>│                    │                     │
  │                           │                     │  x402 verify       │                     │
  │                           │                     │  → live facilitator│                     │
  │                           │                     │                    │                     │
  │                           │                     │  robot/tunnel/action                     │
  │                           │                     │───────────────────>│                     │
  │                           │                     │                    │  execute task       │
  │                           │                     │                    │────────────────────>│
  │                           │                     │                    │                     │
  │                           │                     │                    │  robot/tunnel/result│
  │                           │                     │<───────────────────│                     │
  │                           │                     │                    │                     │
  │                           │  202 { action_id }  │                    │                     │
  │<──────────────────────────│                     │                    │                     │
  │                           │                     │                    │                     │
  │                           │                     │  settle (on success only)                 │
  │                           │                     │  0.001 USDC on Base Sepolia               │
```

---

## Results

| Metric | MuJoCo | PyBullet | Webots R2025a |
|---|---|---|---|
| Status | **success** | **success** | **success** |
| Targets reached | **3 / 3** | **3 / 3** | **3 / 3** |
| Mean EE error | **9.54 mm** | **12.18 mm** | **8.98 mm** |
| Max EE error | 13.49 mm | 19.79 mm | 12.33 mm |
| Min pelvis height | **0.908 m** | **0.940 m** | **0.898 m** |
| Falls | **0** | **0** | **0** |
| Collisions | **0** | **0** | **0** |
| Duration | 4.61 s | 4.98 s | 7.70 s |

**Cross-engine spread: 3.20 mm** (limit: 50 mm) → **PASS**

---

## Payment Security

| Attack | Defense | Result |
|---|---|---|
| No payment | x402 gate returns 402 | **blocked** |
| Wrong amount | Protocol check → 400 | **blocked** |
| Malformed txHash | Protocol check → 400 | **blocked** |
| Forged authorization | Live facilitator rejects | **blocked** |
| Replay | Idempotency store | **blocked** |
| Failed execution | Settlement gate | **not settled** |
| Silent robot | Timeout watcher | **not settled** |

**Invariant:** A failed execution is never settled. The USDC contract confirms this.

---

## Settlement Evidence

| Field | Value |
|---|---|
| Action | `act-paid-de66513f791b` |
| Tx | [`0x2b3b71d0…c0f39`](https://sepolia.basescan.org/tx/0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39) |
| Block | 45706216 |
| Amount | 0.001 USDC |
| Network | Base Sepolia |
| Binding | `keccak256(action_id)` = authorization nonce |

---

## Technical Stack

| Component | Technology |
|---|---|
| Robot | Boston Dynamics Atlas v4 (MIT, fetched not vendored) |
| Simulator | MuJoCo 3.11, PyBullet, Webots R2025a |
| Controller | Damped-least-squares resolved-rate IK |
| Transport | Zenoh (peer mode) |
| Payment | x402 protocol, EIP-3009, USDC |
| Blockchain | Base Sepolia |
| Tunnel | Go, gorilla/websocket |
| Bridge | Python, MuJoCo bindings |
| CI | GitHub Actions |

---

## Key Files

| File | What it does |
|---|---|
| `control_core.py` | Closed-loop DLS controller — re-solves IK each tick |
| `kinematics.py` | Shared Jacobian + gravity from URDF |
| `task.py` | 3-target shelf inspection state machine |
| `handlers.go` | Go tunnel settlement gate |
| `handlers_test.go` | 8 tests proving settlement invariants |
| `real_paid_run.py` | Live facilitator → robot → on-chain settlement |
| `sim2sim.py` | Cross-engine comparison and verdict |

---

## Reproduce

```bash
# Full reproduction from clean checkout
pip install -r projects/robopay/bridge/boston_dynamics/atlas_bridge/requirements.txt
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.download_atlas_model

# 116 tests
python -m pytest projects/robopay/bridge/boston_dynamics.atlas_bridge/tests -q

# MuJoCo execution
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.runner

# PyBullet execution
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.pybullet_runner

# Cross-engine comparison
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.sim2sim

# Full demo (Zenoh transport)
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.demo_tunnel

# Settlement evidence verification
python -m projects.robopay.bridge.boston_dynamics.atlas_bridge.settlement_evidence
```

---

## Links

| Resource | URL |
|---|---|
| PR #120 | https://github.com/fabricfoundation/RoboPay/pull/120 |
| RoboPay repo | https://github.com/fabricfoundation/RoboPay |
| Settlement tx | https://sepolia.basescan.org/tx/0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39 |
| Author | https://github.com/EslaM-X |

---

## License

MIT — see [LICENSE](LICENSE)

Atlas v4 model: MIT — see [NOTICE.md](projects/robopay/NOTICE.md)

---

<div align="center">

**Built in Egypt. Open to the world.**

</div>
