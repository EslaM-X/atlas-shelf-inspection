<div align="center">

<p align="center">
  <img src="https://avatars.githubusercontent.com/u/0?size=200" width="120" alt="EslaM-X">
</p>

# EslaM-X Engineering

**Open-source research in cryptographic verification and autonomous robotics.**

[![GitHub](https://img.shields.io/badge/GitHub-EslaM-X-181717?logo=github&logoColor=white)](https://github.com/EslaM-X)
[![Email](https://img.shields.io/badge/Contact-EslaM--X%40users.noreply.github.com-blue)](mailto:EslaM-X@users.noreply.github.com)
[![Made in Egypt](https://img.shields.io/badge/Made%20with%20%E2%9D%A4%20in-Egypt-white?logo=github&logoColor=white)](https://github.com/EslaM-X)

---

</div>

## Mission

Building infrastructure that makes software execution verifiable and autonomous systems economically accountable. Two parallel research tracks:

1. **Cryptographic proof protocols** — turning CI/CD pipelines into independently verifiable evidence chains
2. **Payment-gated autonomous robotics** — making robots economically accountable for every action they execute

---

## Research Tracks

### Track A — Cryptographic Verification Protocol

<div align="center">

| | |
|---|---|
| **Repository** | [EslaM-X/proofx](https://github.com/EslaM-X/proofx) |
| **Status** | v0.4.0 stable, protocol frozen |
| **Language** | Go, Rust, WebAssembly |

</div>

**The problem:** Software supply chains rely on checksums that prove integrity but not provenance. You know a binary hasn't changed, but you don't know who built it or from what source.

**The solution:** ProofX turns software execution into a cryptographic proof that anyone can independently verify. A proof binds source code, CI run, build artifacts, and claims into a Merkle tree signed with ed25519.

**Key achievements:**
- 102-case cross-language conformance suite
- Three independent implementations: Go (102/102), WASM (102/102), Rust (63/63)
- 14 adversarial attack scenarios documented and tested
- Protocol v2 with execution-centric evidence model
- Backward-compatible with v0.3 proofs

**Adoption status:**

| PR | Repository | Sector | Status |
|---|---|---|---|
| [#729](https://github.com/rhysd/actionlint/pull/729) | [rhysd/actionlint](https://github.com/rhysd/actionlint) | DevOps/CI | Open — awaiting maintainer |
| [#2681](https://github.com/buildpacks/pack/pull/2681) | [buildpacks/pack](https://github.com/buildpacks/pack) | DevOps/CI | Open — awaiting review |

---

### Track B — Payment-Gated Autonomous Robotics

<div align="center">

| | |
|---|---|
| **Repository** | [fabricfoundation/RoboPay](https://github.com/fabricfoundation/RoboPay) |
| **PR** | [#120](https://github.com/fabricfoundation/RoboPay/pull/120) |
| **Language** | Python, Go, Solidity |

</div>

**The problem:** Autonomous robots operate without economic accountability. There is no protocol that ties payment to verified execution — a robot can accept payment and deliver nothing, or execute a different action than what was paid for.

**The solution:** RoboPay implements x402 payment-gated robot control. A client discovers a robot's skills and price, signs an EIP-3009 authorization, and only after verified execution does settlement occur on-chain. Every payment is bound to a specific action via `keccak256(action_id)` as the authorization nonce.

**Key achievements:**
- Boston Dynamics Atlas v4 — free-standing shelf inspection across MuJoCo, PyBullet, and Webots
- Real USDC settlement on Base Sepolia (0.001 USDC per action)
- Payment-validated action over real Zenoh transport
- 3/3 targets on all three simulation engines
- 3.20 mm cross-engine spread (limit: 50 mm)
- Go tunnel with x402 middleware gating unpaid and forged actions
- 116 tests including settlement, idempotency, and forge-refusal

**End-to-end flow:**

```
Client → Fabric Relay → Go Tunnel → x402 Verification → Zenoh → Robot → Settlement
         (discovery)    (gating)     (live facilitator)   (transport) (MuJoCo) (on-chain)
```

**Evidence:**

| Artifact | What it proves |
|---|---|
| `real-paid-run.json` | Live facilitator accepts → robot 3/3 → 0.001 USDC settled |
| `fabric-relay-e2e.json` | Full path through hosted relay, discovery, pricing, execution, settlement |
| `sim2sim-validation.json` | Cross-engine agreement: 3.20 mm spread |
| `go-tunnel-e2e-evidence.json` | Unpaid and forged actions refused by Go tunnel |

---

## Engineering Principles

### Proof of Work, Not Proof of Marketing

Every claim is backed by a committed artifact. Every artifact is checksummed. Every checksum survives a clone. The evidence speaks; the README does not need to.

### Additive Only

External contributions are additive. No existing steps modified. No new permissions required. No README spam. No badge changes. If the contribution is removed, nothing breaks.

### Differential Verification

The same proof is verified by three independent implementations. Zero differences means the protocol is implemented consistently. The conformance suite is the contract.

### Economic Accountability

Every paid robot action is bound to its settlement on-chain. A failed execution is never settled. The token contract itself confirms whether an authorization was spent.

---

## Technical Stack

| Domain | Technologies |
|---|---|
| **Languages** | Go 1.23+, Rust 1.97, Python 3.11+, Solidity |
| **Cryptography** | SHA-256, ed25519, Merkle trees, keccak256 |
| **Robotics** | MuJoCo, PyBullet, Webots R2025a, Zenoh |
| **Blockchain** | Base Sepolia, USDC (EIP-3009), x402 protocol |
| **CI/CD** | GitHub Actions, differential testing |
| **Build** | WASM (TinyGo), cross-platform binaries |

---

## Publications & Evidence

### ProofX
- [Protocol Specification](https://github.com/EslaM-X/proofx/blob/main/docs/SPEC.md)
- [Cryptography Specification](https://github.com/EslaM-X/proofx/blob/main/docs/CRYPTOGRAPHY.md)
- [Threat Model](https://github.com/EslaM-X/proofx/blob/main/docs/THREAT_MODEL.md)
- [Attack Scenarios](https://github.com/EslaM-X/proofx/blob/main/security/ATTACK_SCENARIOS.md)

### RoboPay
- [Atlas Shelf Inspection PR](https://github.com/fabricfoundation/RoboPay/pull/120)
- [Validation Report](https://github.com/fabricfoundation/RoboPay/pull/120/files#diff-validation-report)
- [Settlement Evidence](https://sepolia.basescan.org/tx/0x2b3b71d0ce18554a4927e1145a704359bad35c209f632dc414926b995aac0f39)

---

## Roadmap

### Completed
- [x] ProofX v0.4.0 protocol frozen and released
- [x] 102-case cross-language conformance suite
- [x] Independent Rust verifier
- [x] Attack lab with 14 adversarial scenarios
- [x] RoboPay Atlas shelf inspection with real settlement
- [x] Cross-engine sim-to-sim validation (3 engines)

### In Progress
- [ ] ProofX external adoption (PRs open on actionlint, buildpacks/pack)
- [ ] RoboPay PR #120 review and merge
- [ ] v0.5 protocol design based on adoption feedback

### Planned
- [ ] Independent security review
- [ ] SDK releases (Go/JS/Python)
- [ ] Multi-robot payment profiles
- [ ] Production settlement on mainnet

---

## Contact

| Channel | Link |
|---|---|
| GitHub | [github.com/EslaM-X](https://github.com/EslaM-X) |
| ProofX | [github.com/EslaM-X/proofx](https://github.com/EslaM-X/proofx) |
| Issues | [ProofX Issues](https://github.com/EslaM-X/proofx/issues) |
| Discussions | [ProofX Discussions](https://github.com/EslaM-X/proofx/discussions) |

---

## License

This repository is released under the [MIT License](LICENSE).

Individual projects retain their own licenses:
- [ProofX](https://github.com/EslaM-X/proofx/blob/main/LICENSE) — MIT
- RoboPay contributions — see [fabricfoundation/RoboPay](https://github.com/fabricfoundation/RoboPay)

---

<div align="center">

**Built in Egypt. Open to the world.**

</div>
