# Skill Catalog — Atlas Shelf Inspection

**Profile:** `boston-dynamics.atlas.mujoco-pybullet-webots-shelf-inspection.v1`

---

## Skills

### inspect_shelf

```json
{
  "skill_id": "inspect_shelf",
  "description": "Three-target shelf inspection using closed-loop DLS resolved-rate control",
  "price_usdc": 1000,
  "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
  "network": "eip155:84532",
  "params": {
    "timeout_seconds": {
      "type": "integer",
      "minimum": 5,
      "maximum": 60,
      "default": 30
    }
  },
  "targets": 3,
  "tolerance_mm": 10,
  "hold_steps": 250,
  "engines": ["mujoco", "pybullet", "webots"]
}
```

### stop

```json
{
  "skill_id": "stop",
  "description": "Emergency stop — halts execution immediately",
  "price_usdc": 0,
  "params": {}
}
```

---

## Pricing

| Skill | Price | Asset | Network |
| --- | --- | --- | --- |
| `inspect_shelf` | 0.001 USDC (1000 raw) | USDC | Base Sepolia |
| `stop` | Free | — | — |

The price is discovered at runtime via `GET /skills`, not hardcoded in the client. The payment is built from the amount the relay quotes in its 402 response.

---

## Parameter Bounds

| Parameter | Minimum | Maximum | Default | Enforcement |
| --- | --- | --- | --- | --- |
| `timeout_seconds` | 5 | 60 | 30 | Bridge rejects out-of-bounds before execution |

A request with `timeout_seconds: 1` is refused by the bridge with `INVALID_DURATION` and never reaches the simulator.
