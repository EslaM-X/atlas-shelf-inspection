<div align="center">

# RoboPay — Boston Dynamics Atlas Shelf Inspection

**Payment-gated, policy-driven shelf inspection on a free-standing Boston Dynamics Atlas v4, executed through the RoboPay → Zenoh → simulator path and validated across MuJoCo, PyBullet and Webots R2025a from one pinned robot description.**

[![PR #120](https://img.shields.io/badge/PR-%23120-open-blue)](https://github.com/fabricfoundation/RoboPay/pull/120)
[![Tier 1](https://img.shields.io/badge/Tier-1-brightgreen)](https://github.com/fabricfoundation/RoboPay/pull/120)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

---

</div>

## TL;DR

- **Boston Dynamics Atlas v4** — pinned, MIT-licensed, fetched not vendored
- **3-target** shelf inspection, each held before it counts
- **Closed-loop** damped-least-squares resolved-rate controller
- **Free-standing** — no weld, no clamp, no external support
- **MuJoCo + PyBullet + Webots R2025a** from one URDF
- **3/3 targets on all three engines**
- **0** shelf collisions · **0** falls
- **Sim-to-sim PASS**, 3.20 mm spread across engines
- **Payment-validated action** over real Zenoh transport, correlated by `action_id`
- **x402 safety**: unpaid / invalid / malformed-hash / replay **never reach Zenoh**
- **Live x402 facilitator** refuses a forged authorization — proven, fails closed
- **Durable idempotency** — one actuation per key, across restarts
- **Real paid action**: live facilitator accepts → robot 3/3 → **0.001 USDC settled**
- **Settlement bound** to `action_id` by `keccak256(action_id)`, checkable on chain

---

## End-to-End Flow

```
Payment-validated action request
    ↓
x402 verification  ── rejected ─→ 402/400, never published, simulator untouched
    ↓
Zenoh: robot/tunnel/action
    ↓
Atlas bridge  (skill + parameter contract)
    ↓
inspect_shelf controller
    ↓
MuJoCo / PyBullet / Webots
    ↓
Zenoh: robot/tunnel/result
    ↓
correlation by action_id
    ↓
settlement — only on success
```

---

## Controller

```
STAND → REACH → VERIFY  ×3 targets  → RETURN → DONE
          ▲        │
          └────────┘   hold broken → re-converge
```

Each control tick re-reads the measured end-effector pose and the measured joint configuration, then solves one damped-least-squares resolved-rate step on the right-arm chain. A target only counts after the hand holds it inside tolerance for 250 consecutive steps. No recorded trajectory is used anywhere.

Atlas is never welded, clamped or externally supported. The fall check uses real standing height — 0.70 m against a 0.911 m stance — not floor contact.

---

## Results

| Metric | MuJoCo | PyBullet | Webots R2025a |
| --- | --- | --- | --- |
| Status | **success** | **success** | **success** |
| Targets reached and held | **3 / 3** | **3 / 3** | **3 / 3** |
| Mean end-effector error | **9.54 mm** | **12.18 mm** | **8.98 mm** |
| Max end-effector error | 13.49 mm | 19.79 mm | 12.33 mm |
| Min pelvis height (fall below 0.70 m) | **0.9084 m** | **0.9395 m** | **0.8982 m** |
| Falls | **0** | **0** | **0** |
| Shelf collisions | **0** | **0** | **0** |
| Episode duration | 4.61 s | 4.98 s | 7.70 s |

Cross-engine agreement: mean-error spread **3.20 mm** (limit 50 mm), duration spread **3.10 s** (limit 5.0 s) → sim-to-sim verdict **PASS**.

---

## The Robot

Atlas v4 is **fetched, never vendored**:

| | |
| --- | --- |
| Upstream | [`openai/roboschool`](https://github.com/openai/roboschool) |
| Commit | `d32bcb2b35b94168b5ce27233ca62f3c8678886f` |
| File | `atlas_description/urdf/atlas_v4_with_multisense.urdf` |
| License | **MIT** — see [`NOTICE.md`](NOTICE.md) |
| Mesh assets committed | **none** — upstream collision geometry is analytic |

```
atlas_v4_with_multisense.urdf   ← one pinned description
        │
        ├── MuJoCo    (URDF → MJCF at load time)
        ├── PyBullet  (loaded directly)
        └── Webots    (URDF → PROTO + world at setup time)
```

The Jacobian **and** the gravity feedforward come from that URDF via `kinematics.py` rather than from each engine, so the controller is identical everywhere. Tests pin both against MuJoCo's independently computed values (Jacobian within `5e-4`, measured worst case `2.4e-4`; gravity model within `1e-3` N·m).

Actuator addressing is read out of the compiled model and validated against the URDF's own effort limits; any drift raises immediately.

---

## Payment

One price and one asset address, declared once and asserted against every consumer:

| | |
| --- | --- |
| Price | **0.001 USDC** = `1000` raw units |
| Asset | USDC [`0x036CbD53…3dCF7e`](https://sepolia.basescan.org/token/0x036CbD53842c5426634e7929541eC2318f3dCF7e) |
| Network | Base Sepolia — `eip155:84532` |

### On-Chain Settlement

| Field | Value |
| --- | --- |
| Action | `act-paid-de66513f791b` |
| Settlement tx | [`0x2b3b71d0…c0f39`](https://sepolia.basescan.org/tx/0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39) |
| Status | **success** (`0x1`), block **45706216** |
| Event | `Transfer(address,address,uint256)` on the USDC contract |
| Amount | **0.001 USDC** (`1000` raw) — the price the catalogue publishes |
| Payer | [`0xa0597a74…Fc2Dc`](https://sepolia.basescan.org/address/0xa0597a74f3C3F33797007495bc3Dc676F10Fc2Dc) |
| Payee | [`0x7b916325…C3e8`](https://sepolia.basescan.org/address/0x7b9163254A21b249a0D3E34300fC81BB0A43C3e8) |
| Binding | authorization nonce = `keccak256(action_id)` |

---

## Through the Hosted Fabric Relay

`fabric-relay-e2e.json` is the run where every component is the real one — the hosted relay at `api.fabric.foundation`, this repository's Go tunnel dialled out to it over WSS, the live facilitator, Zenoh, and MuJoCo:

```
client
  ↓  hosted Fabric relay   https://api.fabric.foundation/api/core
  ↓  Go tunnel (this repo, dialled out over WSS)
  ↓  x402 middleware → live facilitator
  ↓  Zenoh robot/tunnel/action → Atlas bridge → MuJoCo
  ↓  Zenoh robot/tunnel/result
  ↓  relay terminal status, correlated by action_id
  ↓  0.001 USDC settled on Base Sepolia
```

| Step | Result |
| --- | --- |
| Robot discovery | `GET /robots/{id}/skills` → **200**, robot connected |
| Skill discovery | `inspect_shelf`, `stop` |
| Price discovery | **0.001 USDC** — read from the response, not assumed |
| Unpaid action | **402** from the relay, carrying payment requirements |
| Quoted amount | `1000` raw, matching the discovered price |
| Paid action | **202 accepted** immediately, carrying `action_id` and a status URL |
| Execution | **3/3 targets** |
| Terminal status | `succeeded`, correlated by `action_id`, read back from the relay |
| Settlement | after the result — [`0xfd9eda75…1e6940`](https://sepolia.basescan.org/tx/0xfd9eda75ddc6c6f979eb2571e6e85ef3a6f50d670f3f8ad252107723e21e6940), block 45714728 |
| Token's own record | `authorizationState(...) = true` — the authorization was spent |
| Binding | on-chain nonce = `keccak256("atlas-inspect-1787197727")` |

---

## Payment Gates — Two Layers

1. **Protocol checks** — amount, asset, network, expiry, settlement-reference shape, replay. Cheap, and they reject the obvious cases first.
2. **Facilitator verification** — the only check that can tell a real authorization from a well-formed forgery, because only the facilitator recovers the signer.

The forgery case is proven against the **live** facilitator: a payload with the right amount, asset, network and a perfectly shaped signature is refused with `invalid_exact_evm_signature`. A companion test asserts the uncomfortable half — the protocol checks alone *do* accept that same payload, which is precisely why the facilitator layer exists. Verification **fails closed**: an unreachable facilitator is a rejection, never an approval.

---

## Request/Response Matrix

| Request | Verified | Published to Zenoh | Executed | Settled |
| --- | --- | --- | --- | --- |
| No payment | no — 402 | **no** | no | no |
| Wrong amount | no — 400 | **no** | no | no |
| Malformed `txHash` | no — 400 | **no** | no | no |
| Valid receipt | protocol checks pass | yes | **3/3 targets** | recorded |
| Replayed receipt | no — 400 | **no** | no | no |
| Undeclared parameter | yes | yes | refused by the bridge | no |
| Forged authorization | no — live facilitator | **no** | no | no |

---

## Idempotency

| Repeat | Outcome |
| --- | --- |
| Same key, same request | `duplicate` — recorded outcome replayed, robot does not move |
| Same key, different parameters | refused, `IDEMPOTENCY_PARAMS_CONFLICT` |
| Same key, different payment | refused, `IDEMPOTENCY_PAYMENT_CONFLICT` |
| Same key after a restart | still **one** actuation |

---

## Settlement Invariants

| Case | Test |
| --- | --- |
| Execution reports failure | `TestAFailedEpisodeIsNeverSettled` · `test_valid_payment_failure_no_settlement` |
| The robot never answers — execution times out | `TestASilentRobotIsNeverSettled` |
| The skill raises mid-episode | `test_execution_exception_no_settlement` |
| A payment is replayed | `test_replay_rejected` · `test_no_double_settle` |
| Execution succeeds | settles **exactly once** — `TestSettlementFollowsSuccess` · `test_settle_on_success_only` |
| The action never reaches Zenoh | `TestAFailedPublishLeavesNoWaiterBehind` — 502, no waiter, no settlement |

---

## Evidence

| Artifact | What it shows |
| --- | --- |
| `atlas-paid-action.gif` | **One continuous pass**: 402 → signed authorization → 202 → Zenoh → execution → result → settlement, beside the episode it paid for |
| `atlas-shelf-inspection.gif` | The annotated episode — same run as the metrics |
| `mujoco-inspection-episode.json` | Full MuJoCo metrics, per-target accuracy |
| `pybullet-inspection-episode.json` | Same task on the second engine |
| `webots-inspection-episode.json` | Same task on Webots R2025a |
| `sim2sim-validation.json` | Cross-engine comparison and verdict |
| `reach-envelope.json` | 36-probe sweep the shelf geometry is derived from |
| `go-tunnel-e2e-evidence.json` | Unpaid and forged actions refused by this repo's Go tunnel |
| `fabric-relay-e2e.json` | **The whole path**: hosted relay, discovery, priced 402, paid action, Atlas 3/3, relay status, 0.001 USDC settled |
| `fabric-relay-failure.json` | The unhappy path: refused by declared bounds, answered 202, reported failed — never charged |
| `real-paid-run.json` | **One real paid action**: live facilitator accepts, robot 3/3, 0.001 USDC settled |
| `tunnel-e2e-evidence.json` | Payment-validated action over the real Zenoh transport |
| `onchain-settlement.json` | Base Sepolia settlement, re-read from a public RPC |

---

## Reproduce

```bash
pip install -r bridge/boston_dynamics/atlas_bridge/requirements.txt
python -m bridge.boston_dynamics.atlas_bridge.download_atlas_model
python -m pytest bridge/boston_dynamics/atlas_bridge/tests -q     # 116 passed
python -m bridge.boston_dynamics.atlas_bridge.runner              # exit 0 only on success
python -m bridge.boston_dynamics.atlas_bridge.pybullet_runner
python -m bridge.boston_dynamics.atlas_bridge.sim2sim
python -m bridge.boston_dynamics.atlas_bridge.demo_tunnel
python -m bridge.boston_dynamics.atlas_bridge.settlement_evidence
```

---

## Known Boundaries

- Simulator-only. No physical Atlas validation is claimed.
- Webots execution evidence is a local headless run, not GitHub-runner execution.
- Testnet settlement on Base Sepolia; the USDC has no monetary value.
- The paid run was executed by an operator-held wallet; no key material lives in this repository.
- The relay is the hosted service; its source is not in this repository.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client (Python)                       │
│  discovery → 402 → EIP-3009 sign → POST /action         │
└───────────────────────┬─────────────────────────────────┘
                        │ WSS
┌───────────────────────▼─────────────────────────────────┐
│              Fabric Relay (hosted)                       │
│  api.fabric.foundation/api/core                         │
└───────────────────────┬─────────────────────────────────┘
                        │ WSS
┌───────────────────────▼─────────────────────────────────┐
│              Go Tunnel (this repo)                       │
│  POST /action → x402 gate → settlement watcher          │
│  GET /robot, /skills, /action/:id/status                │
└───────────────────────┬─────────────────────────────────┘
                        │ Zenoh
┌───────────────────────▼─────────────────────────────────┐
│           Atlas Bridge (Python)                          │
│  skill contract → task.py → control_core.py              │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│           Simulator (MuJoCo / PyBullet / Webots)         │
│  atlas_v4_with_multisense.urdf → execution               │
└─────────────────────────────────────────────────────────┘
```

---

## Tasks Completed

- [x] Pin the Atlas v4 model with its licence and provenance
- [x] Remove every vendored Atlas mesh asset
- [x] Validate actuator mapping against the pinned URDF
- [x] Fail loudly on any actuator or effort-limit drift
- [x] Derive one shared URDF kinematics for all engines
- [x] Validate the Jacobian independently against MuJoCo
- [x] Validate the shared gravity feedforward against MuJoCo
- [x] Measure the free-standing reach envelope
- [x] Constrain the inspection targets to the validated envelope
- [x] Implement the closed-loop DLS inspection controller
- [x] Prove no recorded trajectory is used
- [x] Execute three sequential inspection targets
- [x] Require a hold before a target counts as reached
- [x] Detect falls from standing height, not floor contact
- [x] Detect shelf collisions
- [x] Validate MuJoCo execution
- [x] Validate PyBullet execution
- [x] Validate Webots R2025a execution
- [x] Complete cross-engine sim-to-sim validation
- [x] Wire the tunnel bridge to the registered skill and profile
- [x] Carry a payment-validated action over the real Zenoh transport end to end
- [x] Correlate every result by `action_id` and the tunnel's identity fields
- [x] Refuse unpaid, invalid, malformed-hash and replayed receipts
- [x] Verify payments against the live x402 facilitator, failing closed
- [x] Prove a forged authorization is refused before anything executes
- [x] Make idempotency durable — one actuation per key, across restarts
- [x] Accept both snake_case and camelCase tunnel envelopes
- [x] Gate an action through this repository's own Go tunnel and x402 middleware
- [x] Make the idempotency claim atomic and prove it under concurrency
- [x] Require the identity fields the tunnel correlates on, with no inference
- [x] Emit `params_hash` in the published `sha256:<hex>` form for every action
- [x] Record the terminal outcome so a duplicate reports how the first run ended
- [x] Classify each refused payment by its actual reason in the ledger
- [x] Add safe-stop behaviour and prove a stopped run never settles
- [x] Prove a failed execution never settles
- [x] Carry a paid action through the **hosted Fabric relay** end to end
- [x] Discover the robot, its skills and its price through the relay before paying
- [x] Build the payment from the discovered price rather than a constant
- [x] Read the terminal result back from the relay, correlated by `action_id`
- [x] Show a refused action reported as `failed` through the relay, with its real reason
- [x] Enforce the catalogue's declared parameter bounds on a paid action
- [x] Settle only after the robot reports success — proven on chain in both directions
- [x] Prove a refused action was never charged, from the token contract's own record
- [x] Refuse an uncorrelatable action **before** anything reaches Zenoh, and test it
- [x] Make the published `functions.yaml` contract match every route's real responses
- [x] Verify in CI that the settlement is bound to the action it paid for
- [x] Fail that check on a wrong amount, asset, payer or payee, not just a wrong binding
- [x] Make the skill contract, README, manifest and report agree on every status code and amount
- [x] Pin MuJoCo to the build the evidence was recorded on, after CI observed the drift
- [x] Build the Go tunnel in CI from a clean checkout and run its payment-contract tests there
- [x] Record the payment and the robot in a single continuous pass over one action
- [x] Refuse an action missing **any** identity field before it reaches Zenoh, and test each one
- [x] Advertise the payee the robot is paid at, and hold it to the configured one in test
- [x] Document wallet setup, testnet configuration and key handling in the README
- [x] Demonstrate a live facilitator-verified positive paid action
- [x] Bind the accepted EIP-3009 authorization and its settlement to the executed `action_id`
- [x] Settle the declared 0.001 USDC price on Base Sepolia, after execution only
- [x] Measure end-effector speed against the hand's own displacement
- [x] Declare one price and one asset address across the whole profile
- [x] Verify the real Base Sepolia settlement from a public RPC
- [x] Publish evidence artefacts with checksums that survive a clone
- [x] Publish a machine-readable skill catalogue for discovery and pricing
- [x] Keep reproduction read-only — no command rewrites committed evidence
- [x] Add reproducible CI covering task, payment, tunnel and settlement
- [x] Add a reviewer-facing annotated GIF
- [x] Document the operational runbook and clean-checkout reproduction

---

<div align="center">

**Built by [EslaM-X](https://github.com/EslaM-X) · [PR #120](https://github.com/fabricfoundation/RoboPay/pull/120)**

</div>
