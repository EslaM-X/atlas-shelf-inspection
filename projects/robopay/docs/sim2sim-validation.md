# Sim-to-Sim Validation

## Overview

The same robot description, task, and controller are executed on three physics engines. Cross-engine agreement is measured and bounded.

---

## Engines

| Engine | Version | Integration |
| --- | --- | --- |
| MuJoCo | 3.11 | URDF → MJCF at load time |
| PyBullet | latest | URDF loaded directly |
| Webots | R2025a | URDF → PROTO + world at setup time |

---

## Shared Components

| Component | Source | Notes |
| --- | --- | --- |
| Robot description | `atlas_v4_with_multisense.urdf` | Pinned, fetched, SHA-256 verified |
| Kinematics | `kinematics.py` | Jacobian + gravity from URDF, not engine |
| Controller | `control_core.py` | DLS resolved-rate, re-solved each tick |
| Task | `task.py` | 3-target shelf inspection, hold-based |

---

## Results

| Metric | MuJoCo | PyBullet | Webots | Spread | Limit |
| --- | --- | --- | --- | --- | --- |
| Status | success | success | success | — | — |
| Targets | 3/3 | 3/3 | 3/3 | 0 | — |
| Mean EE error | 9.54 mm | 12.18 mm | 8.98 mm | 3.20 mm | 50 mm |
| Max EE error | 13.49 mm | 19.79 mm | 12.33 mm | 6.30 mm | — |
| Min pelvis height | 0.908 m | 0.940 m | 0.898 m | 0.042 m | — |
| Falls | 0 | 0 | 0 | 0 | 0 |
| Collisions | 0 | 0 | 0 | 0 | 0 |
| Duration | 4.61 s | 4.98 s | 7.70 s | 3.10 s | 5.0 s |

---

## Verdict

**PASS**

- Mean-error spread: **3.20 mm** (limit 50 mm) ✅
- Duration spread: **3.10 s** (limit 5.0 s) ✅
- Falls: **0** across all engines ✅
- Collisions: **0** across all engines ✅

---

## Why the Numbers Differ

Only the joint servo differs between engines:

| Engine | Servo Model |
| --- | --- |
| MuJoCo | Explicit PD law |
| PyBullet | Implicit servo |
| Webots | Implicit servo (gain raised from default) |

Everything above the servo — kinematics, controller, task — is shared code. The 3.20 mm spread is entirely attributable to servo-level differences, which is the expected behavior for cross-engine validation.

---

## Reproducibility

```bash
# MuJoCo (primary)
python -m bridge.boston_dynamics.atlas_bridge.runner

# PyBullet (cross-check)
python -m bridge.boston_dynamics.atlas_bridge.pybullet_runner

# Webots (local headless)
python -m bridge.boston_dynamics.atlas_bridge.webots_env

# Cross-engine comparison
python -m bridge.boston_dynamics.atlas_bridge.sim2sim
```

MuJoCo runs are deterministic — repeated runs hash identically. The test fingerprints the whole result, not a few metrics.
