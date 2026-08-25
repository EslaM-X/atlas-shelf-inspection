# Boston Dynamics Atlas ΓÇö Tier 1: Paid Multi-Step Shelf Inspection

Payment-gated, policy-driven shelf inspection on a free-standing Boston Dynamics
Atlas v4, executed through the **RoboPay ΓåÆ Zenoh ΓåÆ simulator** path and validated
across MuJoCo, PyBullet and Webots R2025a from one pinned robot description.

![One paid Atlas action, end to end](https://raw.githubusercontent.com/EslaM-X/RoboPay/7ebefaf3904b6029bc14342c13cd29f534df666d/docs/evidence/atlas-paid-action.gif)

*One pass over one action, nothing edited together. The left pane is the real
exchange as it happens ΓÇö skill and price discovered, the unpaid **402**, the
EIP-3009 authorization signed with `keccak256(action_id)` as its nonce, the
**202**, the action on Zenoh, the correlated result, and **0.001 USDC settled**
with its BaseScan link and the token's own confirmation that the authorization
was spent. The right pane is the episode that payment bought, rendered from
inside it: the frames are drawn from a callback on the scored run's own control
loop, so the two panes cannot be two different runs that happened to agree. The
settlement is
[`0x7fc1c1ffΓÇªb5b197`](https://sepolia.basescan.org/tx/0x7fc1c1ff21e535376bf22b3409b751d769dadfdcb2491d3803c5712905b5b197),
block 45737187, and you can reach it without taking my word for the hash ΓÇö
`keccak256("atlas-inspect-1787242629")` is the authorization nonce, which is
enough to find the `AuthorizationUsed` event on USDC and read the transfer off
it.*

![Atlas shelf inspection](https://raw.githubusercontent.com/EslaM-X/RoboPay/7ebefaf3904b6029bc14342c13cd29f534df666d/docs/evidence/atlas-shelf-inspection.gif)

*The task close-up, in MuJoCo. Green spheres are the three targets at their
tolerance radius; the panel is live telemetry. The renderer refuses to write the
GIF unless the annotated episode reproduces the plain one exactly, so the picture
and the numbers cannot disagree.*

## TL;DR

- ≡ƒñû Boston Dynamics **Atlas v4** ΓÇö pinned, MIT-licensed, fetched not vendored
- ≡ƒÄ» **3-target** shelf inspection, each held before it counts
- ≡ƒºá Closed-loop damped-least-squares resolved-rate controller
- ≡ƒºì **Free-standing** ΓÇö no weld, no clamp, no external support
- ≡ƒº¬ **MuJoCo + PyBullet + Webots R2025a** from one URDF
- ≡ƒôè **3/3 targets on all three engines**
- ≡ƒÆÑ **0** shelf collisions ┬╖ ≡ƒºÄ **0** falls
- ≡ƒöü Sim-to-sim **PASS**, 3.20 mm spread across engines
- ≡ƒöî Payment-validated action over the **real Zenoh transport**, correlated by `action_id`
- ≡ƒÆ│ x402 safety: unpaid / invalid / malformed-hash / replay **never reach Zenoh**;
      a failed execution is accepted asynchronously and **never settles**
- ≡ƒ¢í∩╕Å **Live x402 facilitator** refuses a forged authorization ΓÇö proven, fails closed
- ≡ƒöÆ **Durable idempotency** ΓÇö one actuation per key, across restarts
- ≡ƒîÉ **The whole path, nothing stood in for**: hosted Fabric relay ΓåÆ this repo's Go tunnel
      ΓåÆ live facilitator ΓåÆ Zenoh ΓåÆ Atlas 3/3 ΓåÆ relay status ΓåÆ settlement
- ≡ƒöÄ **Robot, skill and price discovery** through the relay ΓÇö the payment is built from the
      price the relay quotes, not from a constant
- ≡ƒÆ░ **One real paid action**: live facilitator accepts ΓåÆ robot 3/3 ΓåÆ **0.001 USDC settled**
- Γ¢ô∩╕Å Settlement **bound to the `action_id`** by `keccak256(action_id)`, checkable on chain
- ≡ƒôª Reproducible from a clean checkout

---

## Tier 1 success criteria

| Requirement | How it is met | Evidence |
| --- | --- | --- |
| Simulator-only | MuJoCo, PyBullet, Webots ΓÇö no hardware anywhere in the repo | [`validation-report.md`](registry/vendors/boston-dynamics/atlas/boston-dynamics.atlas.mujoco-pybullet-webots-shelf-inspection.v1/docs/validation-report.md) |
| Approved simulator | MuJoCo 3.11 primary; Webots R2025a and PyBullet cross-check | `sim2sim-validation.json` |
| Policy / controller triggered | State machine + DLS resolved-rate IK, re-solved from the measured pose each tick | [`control_core.py`](bridge/boston_dynamics/atlas_bridge/control_core.py) |
| Not a replayed trajectory | No recorded trajectory exists; a test feeds two controllers different measured poses and asserts different commands | `test_controller_is_closed_loop_not_a_replayed_trajectory` |
| Not a built-in demo motion | Controller and task written here; the upstream description contributes only the robot | [`task.py`](bridge/boston_dynamics/atlas_bridge/task.py) |
| Simulator state metrics | Per-target end-effector error, targets completed, pelvis height, collisions, duration | `mujoco-inspection-episode.json` |
| Sim-to-Sim validation | Same URDF, task and controller on three engines, bounded spread | `sim2sim-validation.json` |
| Payment-validated action ΓåÆ Zenoh ΓåÆ simulator | Verified action published on `robot/tunnel/action`, executed, answered on `robot/tunnel/result` | `tunnel-e2e-evidence.json` |
| Real Go tunnel gates the action | This repo's tunnel binary refuses unpaid (402) and forged (400) actions; neither reaches the simulator | `go-tunnel-e2e-evidence.json` |
| Forged payment refused | Live x402 facilitator returns `invalid_exact_evm_signature`; nothing executes | `tests/test_facilitator.py` |
| One actuation per payment-validated action | Durable idempotency store, surviving a restart | `tests/test_idempotency.py` |
| Clean, easy-to-understand code | no vendored binaries, one source of truth per concept | this diff |

---

## End-to-end flow

```text
Payment-validated action request
    Γåô
x402 verification  ΓöÇΓöÇ rejected ΓöÇΓåÆ 402/400, never published, simulator untouched
    Γåô
Zenoh: robot/tunnel/action
    Γåô
Atlas bridge  (skill + parameter contract)
    Γåô
inspect_shelf controller
    Γåô
MuJoCo / PyBullet / Webots
    Γåô
Zenoh: robot/tunnel/result
    Γåô
correlation by action_id
    Γåô
settlement ΓÇö only on success
```

`demo_tunnel.py` runs exactly that over the real transport (Zenoh peer mode, no
router required). Every request in the walkthrough:

| Request | Verified | Published to Zenoh | Executed | Settled |
| --- | --- | --- | --- | --- |
| No payment | no ΓÇö 402 | **no** | no | no |
| Wrong amount | no ΓÇö 400 | **no** | no | no |
| Malformed `txHash` | no ΓÇö 400 | **no** | no | no |
| Valid receipt ΓÇá | protocol checks pass | yes | **3/3 targets** | recorded, not on-chain |
| Replayed receipt ΓÇá | no ΓÇö 400 | **no** | no | no |
| Undeclared parameter | yes | yes | refused by the bridge | no |
| Forged authorization | no ΓÇö live facilitator | **no** | no | no |

ΓÇá Replay answers differently on each path, because each one answers at a
different moment:

| Path | Replay response |
| --- | --- |
| Transport demo (`demo_tunnel.py`, Python client over Zenoh) | `400` |
| In-process relay (`demo_e2e.py`, answers synchronously) | `409` |
| Hosted Go tunnel (`demo_fabric_e2e.py`) | `202`, then `DUPLICATE_ACTION` on the status endpoint |

These are three distinct transport and execution paths. All three reject a
replayed execution, and none actuates the robot a second time. The skill
contract names each case separately rather than leaving the difference to be
inferred from the code.

ΓÇá **What the accepting row is, and where the real one lives.** That receipt is
a synthetic hash, so the row shows the bridge's protocol checks passing ΓÇö
amount, format, replay, declared parameters ΓÇö and the action being executed and
answered. **This walkthrough holds no wallet**: it verifies no signature with
the facilitator and moves no value, and its artifact says so
(`payment_verification: protocol_checks_only`, `settlement:
eligible_not_on_chain`). Its job is the transport and the refusal paths; the
forged authorization here is refused by the **live** x402 facilitator.

Accepted live payments are proven separately, twice, and both settle real USDC:
`real-paid-run.json` (facilitator `isValid: true` ΓåÆ 3/3 ΓåÆ 0.001 USDC) and
`fabric-relay-e2e.json` (the same, through the hosted relay). Nothing in this
section should be read as the profile's evidence for a real payment.

Two properties the demo asserts on itself: an unverified payment **never reaches
Zenoh**, and every executed action is answered carrying the originating
`action_id`, `robot_id`, `skill_id`, `params_hash` and `idempotency_key`.

### Through the hosted Fabric relay, with nothing stood in for

`fabric-relay-e2e.json` is the run where every component is the real one ΓÇö the
hosted relay at `api.fabric.foundation`, this repository's Go tunnel dialled out
to it over WSS, the live facilitator, Zenoh, and MuJoCo:

```text
client
  Γåô  hosted Fabric relay   https://api.fabric.foundation/api/core
  Γåô  Go tunnel (this repo, dialled out over WSS)
  Γåô  x402 middleware ΓåÆ live facilitator
  Γåô  Zenoh robot/tunnel/action ΓåÆ Atlas bridge ΓåÆ MuJoCo
  Γåô  Zenoh robot/tunnel/result
  Γåô  relay terminal status, correlated by action_id
  Γåô  0.001 USDC settled on Base Sepolia
```

| Step | Result |
| --- | --- |
| Robot discovery | `GET /robots/{id}/skills` ΓåÆ **200**, robot connected |
| Skill discovery | `inspect_shelf`, `stop` |
| Price discovery | **0.001 USDC** ΓÇö read from the response, not assumed |
| Unpaid action | **402** from the relay, carrying payment requirements |
| Quoted amount | `1000` raw, matching the discovered price |
| Paid action | **202 accepted** immediately, carrying `action_id` and a status URL |
| Execution | **3/3 targets** |
| Terminal status | `succeeded`, correlated by `action_id`, read back from the relay |
| Settlement | after the result ΓÇö [`0xfd9eda75ΓÇª1e6940`](https://sepolia.basescan.org/tx/0xfd9eda75ddc6c6f979eb2571e6e85ef3a6f50d670f3f8ad252107723e21e6940), block 45714728 |
| Token's own record | `authorizationState(...) = true` ΓÇö the authorization was spent |
| Binding | on-chain nonce = `keccak256("atlas-inspect-1787197727")` |

**The price is discovered, not assumed.** The payment is built from the amount
the relay quotes in its 402, and the run asserts that amount equals the price the
catalogue publishes ΓÇö a profile whose advertised price had drifted from what its
tunnel charges would fail that check rather than pass quietly.

**What this changes in the tunnel, stated in full.** Three read-only endpoints
are added ΓÇö `GET /robot`, `GET /skills`, `GET /action/:action_id/status` ΓÇö and
`POST /action` is **extended, not left alone**: it keeps the existing x402
verification boundary, but now refuses an incomplete identity before publishing,
answers `202 Accepted` with an `action_id` and a status URL instead of `200`, and
defers settlement until the correlated result reports success. The stock x402
middleware is replaced by a gate that performs the same `402`/verify half and
hands the settlement to that watcher. Calling this read-only would be
understating it, and the diff would say so.

The status endpoint is not synthesised: the tunnel subscribes to the same
`robot/tunnel/result` topic the simulator publishes on and stores what arrives,
keyed by `action_id`; an unanswered action reads `pending`, a failed one `failed`.
Merged robot-profile branches extend the tunnel the same way.

**A failed action is not paid for, and the token says so.** The same relay,
the same wallet, an action refused by the catalogue's declared bounds
(`atlas-inspect-1787197752`): the tunnel answers **202**, nothing settles, and
`authorizationState(payer, keccak256(action_id))` on the USDC contract is
**false**. "We recorded no transaction hash" is an absence of evidence; the
token's own map of spent authorization nonces is not, and because the nonce is
derived from the action id anyone can recompute it and ask USDC directly.

That on-chain artefact is one action. The invariant behind it ΓÇö an unsuccessful
execution never settles ΓÇö is held at the point where the decision is made, and
each case has a test that fails if the guarantee is removed:

| Case | Test |
| --- | --- |
| Execution reports failure | `TestAFailedEpisodeIsNeverSettled` ┬╖ `test_valid_payment_failure_no_settlement` |
| The robot never answers ΓÇö execution times out | `TestASilentRobotIsNeverSettled` |
| The skill raises mid-episode | `test_execution_exception_no_settlement` |
| A payment is replayed | `test_replay_rejected` ┬╖ `test_no_double_settle` |
| Execution succeeds | settles **exactly once** ΓÇö `TestSettlementFollowsSuccess` ┬╖ `test_settle_on_success_only` |
| The action never reaches Zenoh | `TestAFailedPublishLeavesNoWaiterBehind` ΓÇö 502, no waiter, no settlement |

The Go cases run against the settlement gate itself, which is the component that
decides; the Python cases run against the bridge that produces the result it
decides on. Both suites run in CI on every commit.

This is enforced rather than hoped for, and without giving up the immediate
`202` the tunnel contract promises. The stock x402 middleware settles as soon as
a protected route answers anything under 400, so a `202` would charge the payer
before the robot had run. The tunnel replaces it with a gate that keeps the
`402`/verify half unchanged ΓÇö an unpaid request still gets `402` with the
advertised requirements, a payment the live facilitator rejects still never
reaches the robot ΓÇö and hands a settlement callback to the handler instead of
settling. `POST /action` publishes, answers `202`, and a background watcher
invokes that callback **only** when the correlated result reports success. A
failure or a silent robot leaves the authorization signed and unspent. Eight
tests in `handlers_test.go` hold that with no wallet and no chain, including
that a request with no `action_id` is refused before anything reaches Zenoh.

### Through this repository's own Go tunnel, in isolation

`demo_go_tunnel.py` exercises the refusal paths against the tunnel alone, behind
a minimal stand-in for the relay:

```text
proxy (stands in for the Fabric backend)
    Γåô WebSocket  ws://ΓÇª/api/core/ws/robot?id=atlas-sim-01
Go tunnel  POST /action
    Γåô x402 middleware ΓåÆ live facilitator
Zenoh robot/tunnel/action ΓåÆ Atlas bridge ΓåÆ MuJoCo
```

| Request | The tunnel's answer | Reached the simulator |
| --- | --- | --- |
| No payment | **HTTP 402**, payment requirements advertised | **no** |
| Structurally valid but unsigned authorization | **HTTP 400**, after consulting the facilitator | **no** |

The payment decision there is the tunnel's, not this profile's. Build steps are
in [`TUNNEL_BUILD.md`](bridge/boston_dynamics/atlas_bridge/TUNNEL_BUILD.md).

### Two payment gates, and which one proved what

1. **Protocol checks** ΓÇö amount, asset, network, expiry, settlement-reference
   shape, replay. Cheap, and they reject the obvious cases first.
2. **Facilitator verification** ΓÇö the only check that can tell a real
   authorization from a well-formed forgery, because only the facilitator
   recovers the signer.

The forgery case is proven against the **live** facilitator: a payload with the
right amount, asset, network and a perfectly shaped signature is refused with
`invalid_exact_evm_signature`. A companion test asserts the uncomfortable half ΓÇö
the protocol checks alone *do* accept that same payload, which is precisely why
the facilitator layer exists. Verification **fails closed**: an unreachable
facilitator is a rejection, never an approval.

**The accepting side, proven.** `real-paid-run.json` carries one action the
whole way: an EIP-3009 authorization signed by a funded wallet, accepted by the
**live** facilitator (`isValid: true`), executed on the robot (3/3 targets,
correlated by `action_id`), and settled for **0.001 USDC** ΓÇö the price the
profile declares ΓÇö in [`0x2b3b71d0ΓÇªc0f39`](https://sepolia.basescan.org/tx/0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39), block 45706216.

The settlement is tied to the action rather than merely adjacent to it. EIP-3009
lets the signer choose the authorization nonce, so this run sets it to
`keccak256(action_id)`; the token emits that nonce in `AuthorizationUsed`, so
anyone can recompute it from the action id and confirm this transfer paid for
*this* action:

```
keccak256("act-paid-de66513f791b")   = 0xaa6cf89aΓÇªfd03b47de
AuthorizationUsed nonce, block 45706216 = 0xaa6cf89aΓÇªfd03b47de
```

The transaction was submitted by the facilitator's own address ΓÇö the one its
`/supported` endpoint advertises ΓÇö which is independent evidence it went through
the live facilitator rather than being self-submitted, and why the payer holds
no ETH: under EIP-3009 the payer signs and the facilitator pays the gas.

Order matters and is enforced: `/verify` first, robot second, `/settle` only
because the episode reached every target. A failed episode leaves the
authorization signed and unspent.

The in-process and Zenoh demos remain protocol-level by design ΓÇö they hold no
wallet ΓÇö and their artifacts say so: `settlement: eligible_not_on_chain`,
`settlement_tx_hash: null`. That distinction is enforced by the ledger, not by
wording: `SETTLED` requires a transaction hash **and** its block, and
`test_settled_requires_a_real_transaction` fails if a receipt alone can earn it.

### One actuation per payment-validated action

The idempotency store is keyed on `robot_id + skill_id + idempotency_key`,
records the parameters and a payment fingerprint, and persists to disk:

| Repeat | Outcome |
| --- | --- |
| Same key, same request | `duplicate` ΓÇö recorded outcome replayed, robot does not move |
| Same key, different parameters | refused, `IDEMPOTENCY_PARAMS_CONFLICT` |
| Same key, different payment | refused, `IDEMPOTENCY_PAYMENT_CONFLICT` |
| Same key after a restart | still **one** actuation |

---

## Controller

```text
STAND ΓåÆ REACH ΓåÆ VERIFY  ├ù3 targets  ΓåÆ RETURN ΓåÆ DONE
          Γû▓        Γöé
          ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ   hold broken ΓåÆ re-converge
```

Each control tick re-reads the measured end-effector pose and the measured joint
configuration, then solves one damped-least-squares resolved-rate step on the
right-arm chain. A target only counts after the hand holds it inside tolerance
for 250 consecutive steps. No recorded trajectory is used anywhere.

Atlas is never welded, clamped or externally supported. The fall check uses real
standing height ΓÇö 0.70 m against a 0.911 m stance ΓÇö not floor contact.

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

Cross-engine agreement: mean-error spread **3.20 mm** (limit 50 mm), duration
spread **3.10 s** (limit 5.0 s) ΓåÆ sim-to-sim verdict **PASS**.

Repeated MuJoCo runs hash identically ΓÇö the test fingerprints the whole result,
not a few metrics.

---

## The robot

Atlas v4 is **fetched, never vendored**:

| | |
| --- | --- |
| Upstream | [`openai/roboschool`](https://github.com/openai/roboschool) |
| Commit | `d32bcb2b35b94168b5ce27233ca62f3c8678886f` |
| File | `atlas_description/urdf/atlas_v4_with_multisense.urdf` |
| License | **MIT** ΓÇö see [`NOTICE.md`](bridge/boston_dynamics/atlas_bridge/NOTICE.md) |
| Mesh assets committed | **none** ΓÇö upstream collision geometry is analytic |

```text
atlas_v4_with_multisense.urdf   ΓåÉ one pinned description
        Γöé
        Γö£ΓöÇΓöÇ MuJoCo    (URDF ΓåÆ MJCF at load time)
        Γö£ΓöÇΓöÇ PyBullet  (loaded directly)
        ΓööΓöÇΓöÇ Webots    (URDF ΓåÆ PROTO + world at setup time)
```

The Jacobian **and** the gravity feedforward come from that URDF via
[`kinematics.py`](bridge/boston_dynamics/atlas_bridge/kinematics.py) rather than
from each engine, so the controller is identical everywhere. Tests pin both
against MuJoCo's independently computed values (Jacobian within `5e-4`, measured
worst case `2.4e-4`; gravity model within `1e-3` N┬╖m).

Actuator addressing is read out of the compiled model and validated against the
URDF's own effort limits; any drift raises immediately.

---

## Payment

One price and one asset address, declared once and asserted against every
consumer:

| | |
| --- | --- |
| Price | **0.001 USDC** = `1000` raw units |
| Asset | USDC [`0x036CbD53ΓÇª3dCF7e`](https://sepolia.basescan.org/token/0x036CbD53842c5426634e7929541eC2318f3dCF7e) |
| Network | Base Sepolia ΓÇö `eip155:84532` |

A settlement reference that is not a 32-byte EVM transaction hash is refused with
`MALFORMED_TX_HASH` before anything executes.

### On-chain settlement

| Field | Value |
| --- | --- |
| Action | `act-paid-de66513f791b` |
| Settlement tx | [`0x2b3b71d0ΓÇªc0f39`](https://sepolia.basescan.org/tx/0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39) |
| Status | **success** (`0x1`), block **45706216** |
| Event | `Transfer(address,address,uint256)` on the USDC contract |
| Amount | **0.001 USDC** (`1000` raw) ΓÇö the price the catalogue publishes |
| Payer | [`0xa0597a74ΓÇªFc2Dc`](https://sepolia.basescan.org/address/0xa0597a74f3C3F33797007495bc3Dc676F10Fc2Dc) |
| Payee | [`0x7b916325ΓÇªC3e8`](https://sepolia.basescan.org/address/0x7b9163254A21b249a0D3E34300fC81BB0A43C3e8) |
| Binding | authorization nonce = `keccak256(action_id)` |

`settlement_evidence.py` reads this transaction **out of `real-paid-run.json`**
rather than from a pinned hash, re-reads it from a public RPC, decodes the
`Transfer` log itself, and exits non-zero if it is missing, reverted, carries no USDC
transfer. The workflow runs it on every push.

---

## Evidence

| Artefact | What it shows |
| --- | --- |
| [`atlas-paid-action.gif`](docs/evidence/atlas-paid-action.gif) | **One continuous pass**: 402 ΓåÆ signed authorization ΓåÆ 202 ΓåÆ Zenoh ΓåÆ execution ΓåÆ result ΓåÆ settlement, beside the episode it paid for |
| [`atlas-shelf-inspection.gif`](docs/evidence/atlas-shelf-inspection.gif) | The annotated episode ΓÇö same run as the metrics |
| [`mujoco-inspection-episode.json`](docs/evidence/mujoco-inspection-episode.json) | Full MuJoCo metrics, per-target accuracy |
| [`pybullet-inspection-episode.json`](docs/evidence/pybullet-inspection-episode.json) | Same task on the second engine |
| [`webots-inspection-episode.json`](docs/evidence/webots-inspection-episode.json) | Same task on Webots R2025a |
| [`sim2sim-validation.json`](docs/evidence/sim2sim-validation.json) | Cross-engine comparison and verdict |
| [`reach-envelope.json`](docs/evidence/reach-envelope.json) | 36-probe sweep the shelf geometry is derived from |
| [`go-tunnel-e2e-evidence.json`](docs/evidence/go-tunnel-e2e-evidence.json) | Unpaid and forged actions refused by this repo's Go tunnel |
| [`go-tunnel-e2e-terminal.txt`](docs/evidence/go-tunnel-e2e-terminal.txt) | Terminal transcript of that run |
| [`fabric-relay-e2e.json`](docs/evidence/fabric-relay-e2e.json) | **The whole path**: hosted relay, discovery, priced 402, paid action, Atlas 3/3, relay status, 0.001 USDC settled |
| [`fabric-relay-failure.json`](docs/evidence/fabric-relay-failure.json) | The unhappy path through the same relay: refused by the declared bounds, answered `202`, reported `failed` with its real reason ΓÇö and never charged, confirmed by the token contract |
| [`real-paid-run.json`](docs/evidence/real-paid-run.json) | **One real paid action**: live facilitator accepts, robot 3/3, 0.001 USDC settled, bound to `action_id` |
| [`tunnel-e2e-evidence.json`](docs/evidence/tunnel-e2e-evidence.json) | Payment-validated action over the real Zenoh transport |
| [`tunnel-e2e-terminal.txt`](docs/evidence/tunnel-e2e-terminal.txt) | Terminal transcript of that run |
| [`demo-e2e-evidence.json`](docs/evidence/demo-e2e-evidence.json) | The in-process x402 gate and settlement ledger |
| [`onchain-settlement.json`](docs/evidence/onchain-settlement.json) | Base Sepolia settlement, re-read from a public RPC |

Skill discovery and pricing are published in
[`skill-catalog.json`](registry/vendors/boston-dynamics/atlas/boston-dynamics.atlas.mujoco-pybullet-webots-shelf-inspection.v1/skill-catalog.json),
generated from `skills.yaml` so the two cannot disagree.

Every committed evidence artifact is checksummed in
[`evidence-manifest.yaml`](registry/vendors/boston-dynamics/atlas/boston-dynamics.atlas.mujoco-pybullet-webots-shelf-inspection.v1/docs/evidence/evidence-manifest.yaml),
and stored byte-exact so the recorded hashes still match after a clone.

### Why the shelf is where it is

`reach_envelope.py` drives the arm to a 6 ├ù 6 grid of offsets and records, per
probe, whether the arm converged **and** whether Atlas was still standing:

```text
vert \ fwd   +0.06  +0.12  +0.18  +0.21  +0.24  +0.30
  +0.20       OK     OK     OK     OK    fall   miss
  +0.10       OK     OK     OK    miss   fall   fall
  +0.00       OK     OK     OK     OK     OK     OK
  -0.06       OK     OK     OK     OK     OK     OK
  -0.12       OK     OK     OK     OK    fall   fall
  -0.20      fall   fall   fall   fall   fall   fall
```

The reported envelope is the **largest block where every probe succeeded**
(forward 0.06ΓÇô0.18 m, vertical ΓêÆ0.12ΓÇª+0.20 m, 15/15) ΓÇö not a bounding box around
scattered successes. All three targets sit inside it, and a test fails if one is
ever moved out.

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

Configuration, expected success and failure payloads, safe-stop and
troubleshooting are documented in the
[bridge README](bridge/boston_dynamics/atlas_bridge/README.md#operating-the-bridge).

---

## Tasks

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
- [x] Make idempotency durable ΓÇö one actuation per key, across restarts
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
- [x] Settle only after the robot reports success ΓÇö proven on chain in both directions
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
- [x] Measure end-effector speed against the hand's own displacement, and report
      the speed reached while inspecting separately from the episode peak
- [x] Declare one price and one asset address across the whole profile
- [x] Verify the real Base Sepolia settlement from a public RPC
- [x] Publish evidence artefacts with checksums that survive a clone
- [x] Publish a machine-readable skill catalogue for discovery and pricing
- [x] Keep reproduction read-only ΓÇö no command rewrites committed evidence
- [x] Add reproducible CI covering task, payment, tunnel and settlement
- [x] Add a reviewer-facing annotated GIF
- [x] Document the operational runbook and clean-checkout reproduction

---

## Reviewer notes

- **Webots was validated locally, headless.** It is not available on GitHub's
  runners, so CI verifies that the PROTO and the world still generate from the
  pinned URDF, and `sim2sim` reports the engine as
  `unavailable_no_webots_installation` there, scoring only the engines that ran.
  A missing engine can never turn a failing comparison into a passing one.
- **CI status.** Commit `2b09fc2`, the exact HEAD of this PR, passed every job on
  the author fork ΓÇö
  [run 32399852659](https://github.com/EslaM-X/RoboPay/actions/runs/32399852659),
  which includes the Zenoh tunnel walkthrough, the on-chain settlement
  re-verification, and **the paid action itself** on a clean Ubuntu runner. The
  upstream PR workflow shows `action_required` and awaits maintainer approval,
  which is the normal path for a fork; that is why this PR's own checks tab is
  empty.
- **The payment happens in the run, not beside it.** The `live` job builds the
  tunnel from the checkout, dials the hosted Fabric relay, discovers the robot
  and the price, takes the `402`, signs an EIP-3009 authorization whose nonce is
  `keccak256(action_id)`, runs Atlas, and settles only after the correlated
  result reports success ΓÇö all inside the run being reviewed. In
  [32399852659](https://github.com/EslaM-X/RoboPay/actions/runs/32399852659) that
  was `atlas-inspect-1787248406`, settled in
  [`0x30160596ΓÇªc5a32`](https://sepolia.basescan.org/tx/0x301605968803a459bcf024c8082d414e0501f2d859c71ddbee87b10ee16c5a32)
  at block 45740066. The same job then sends an action the catalogue refuses ΓÇö a
  1-second budget against a declared 5..60 ΓÇö which comes back `failed` with
  `INVALID_DURATION` and settles nothing: `authorizationState` for
  `keccak256("atlas-inspect-1787248427")` is still zero. Both are reachable from
  the action id alone, without trusting the artifact. The job runs on `push` and
  `workflow_dispatch` only, because secrets do not exist in a workflow triggered
  from a fork's pull request; without a credential it still rehearses the whole
  path with `--dry-run`, signing nothing.
- **Where the three engines genuinely differ.** Only the joint servo: MuJoCo
  integrates an explicit PD law, while PyBullet and Webots use their own implicit
  servos, all saturated at the same URDF effort limits. Webots' servo gain had to
  be raised from its default ΓÇö measured, not guessed. Everything above the servo
  is shared code.
- **Every numeric claim above is generated** from a committed evidence artefact.
- **Four demos, and exactly what each one runs.** Only the first reaches the
  hosted relay; the others isolate a layer, and none of them stands in for a
  component the first one exercises.

  | Demo | Hosted relay | Go tunnel | Payment |
  | --- | --- | --- | --- |
  | `demo_fabric_e2e.py` | **real** | **real** | live facilitator, settles on success |
  | `demo_go_tunnel.py` | local proxy | **real** | live facilitator, refusals only |
  | `demo_tunnel.py` | not used ΓÇö Zenoh directly | not used | protocol checks, settles nothing |
  | `real_paid_run.py` | not used ΓÇö Zenoh directly | not used | live facilitator, settles after success |

## Known boundaries

- Simulator-only. No physical Atlas validation is claimed. The robot is the
  description this PR pins ΓÇö `atlas_v4_with_multisense.urdf` from
  `openai/roboschool@d32bcb2`, fetched and SHA-256 verified at build time ΓÇö and
  no claim is made that it is equivalent to, or interchangeable with, any other
  Atlas hardware generation.
- Webots execution evidence is a local headless run, not GitHub-runner execution.
- Testnet settlement on Base Sepolia; the USDC has no monetary value.
- The paid run was executed by an operator-held wallet; no key material lives in
  this repository. Both payer wallets are disposable test wallets, treated as
  compromised, holding nothing of value.
- The relay is the hosted service; its source is not in this repository, so
  `demo_go_tunnel.py` still stands in for it when exercising the tunnel's
  refusal paths in isolation. The positive path does not stand in for anything.
- Testnet only; Base Sepolia USDC has no monetary value. Both payer wallets
  are disposable test wallets, treated as compromised, holding nothing.
- **Robot identity, the outbound client, and the payee** ΓÇö three things that get
  conflated, so, from the code: there is no package called `robotsdk` in this
  repository. The robot-side outbound client it ships is `tunnel/`, and the
  relay connection itself is `tunnel/internal/client.go`, which dials the WSS
  endpoint with `gorilla/websocket`. The same module also carries
  `github.com/unibaseio/aip-go-sdk`, used by `cmd/main.go` and `internal/aipagent`
  for the optional authenticated AIP path ΓÇö not for that transport. `tunnel/` is
  the component the merged Tier-1 profiles use. This profile extends its action
  contract ΓÇö `202`/status semantics and deferred settlement ΓÇö and adds three
  read-only discovery and status endpoints; it changes no transport behaviour. **The authenticating handshake lives in it**: with `AIP_ENABLED=true`
  the tunnel runs `aipauth.EnsureAuth`, receives a bearer token and a wallet
  address, and registers the agent with `Handle` = robot id and `UserID` = that
  wallet. This profile does not bypass it, but the recordings here run with it
  disabled, because `EnsureAuth` drives an interactive browser authorization flow
  that an unattended, reproducible demo cannot perform. So: implemented in the
  component this profile uses, **not exercised in these recordings**, and not
  reimplemented here. The payee half *is* held and tested ΓÇö `GET /robot`, the
  `402` and the settlement all carry the one configured address, and the x402
  matcher refuses a payment whose payee differs. And the binding that *is*
  cryptographic is the settlement to the action: `keccak256(action_id)` as the
  authorization nonce, recorded on chain.
- **This branch predates two changes now on `main`.**
  [#59](https://github.com/fabricfoundation/RoboPay/pull/59) moved the tunnel's
  x402 wiring onto `ginmw.SchemeConfig` with the `evm/exact/server` scheme and
  added `registerTokenAsset` for non-default payment tokens;
  [#126](https://github.com/fabricfoundation/RoboPay/pull/126) added MPP
  alongside x402 on the same `POST /action`. This branch keeps the tunnel API
  that the recorded evidence and CI were produced against, so merging `main`
  into it conflicts in `tunnel/cmd/main.go`,
  `tunnel/internal/handlers/handlers.go` and
  `tunnel/internal/handlers/handlers_test.go` ΓÇö the deferred settlement gate
  sits exactly where those PRs rewrote the middleware call and the handler.
  The Zenoh envelope is not a problem: #126 keeps `payment_payload` and
  `payment_requirements` where they were and only adds `protocol`, and this
  profile's bridge reads those two keys and ignores what it does not know, so
  an envelope produced by the newer tunnel parses here unchanged. That is an architectural
  reconciliation, not a formatting one, and it is deliberately not attempted
  here: doing it would rebuild the component every artifact in this submission
  was recorded against. **No claim is made that the two tunnel implementations
  are drop-in compatible.** The Tier branch this PR targets sits at the commit
  the profile was built on, which is why it merges cleanly there. What has been
  checked is the conflict itself; the work to rebuild the gate on the newer
  scheme registration has not been scoped, and can follow on request.
- **Field names, mapped to the criteria.** The criteria illustrate skill metadata
  as `name` / `priceUSDC` / `paramsSchema`, and a failure envelope as
  `status: "error"` carrying a nested `error.code`. This profile uses the
  repository's existing snake_case forms instead: `skill_id`, `price_usdc` and
  `params` in the catalogue, `robot_id` in the `/skills` envelope, and `status`
  with `result.error_code` in the correlated result. Every concept the criteria
  name is present and carried by the recorded artifacts and the contract tests;
  these are naming choices, not missing data. They are left alone because the
  evidence was recorded against them, and renaming the wire format afterwards
  would put the artifacts and the code out of step.
- **`params_hash` is derived, not carried.** The Zenoh envelope preserves
  `action_id`, `robot_id`, `skill_id`, `idempotency_key` and the payment. The
  bridge computes `params_hash` itself, canonically, from the parameters that
  arrived, and emits it in the `sha256:<hex>` form `execution-mapping.yaml`
  declares ΓÇö including for an empty parameter set. A hash supplied on the wire
  is not consulted. So the hash in every result describes what actually
  executed, and a caller cannot claim one parameter set while sending another;
  but it is a derivation rather than a comparison against a payer-supplied
  value.












































