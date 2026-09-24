# Extending SocIoTy to Threshold ECDSA

This project evaluates whether **threshold ECDSA** can extend the SocIoTy model of "at-home cryptography" beyond lightweight cryptographic primitives.

SocIoTy explores using a smartphone together with smart-home and IoT devices so that sensitive cryptographic operations depend on multiple devices rather than a secret stored in one place. This project studies whether threshold ECDSA can support that model by splitting a signing key across multiple parties and requiring a quorum of those parties to cooperate to produce a valid signature.

The project compares two existing threshold ECDSA implementations:

- [Silence Laboratories DKLS23](https://github.com/silence-laboratories/dkls23)
- [BNB Chain tss-lib](https://github.com/bnb-chain/tss-lib)

The work began as a semester research project for **EE-59903 Advanced Cybersecurity** and was continued after the course to complete the distributed cloud implementation and make the two systems more directly comparable.

---

## Research Question

The main question behind the project is:

> Can real threshold ECDSA implementations support the SocIoTy smart-home signing model when constrained by runtime, memory, communication overhead, and deployment complexity?

The evaluation focuses on:

- distributed key generation / keygen
- distributed signing
- repeated signing using the same generated key
- runtime
- memory / RSS
- network message counts and bytes
- signer subsets
- distributed deployment complexity

---

## Threshold ECDSA

Traditional ECDSA uses one private key held by one signer.

Threshold ECDSA instead splits the private key across multiple parties. No individual party holds the complete private key.

The main parameters are:

- `n` — total number of parties
- `t` — threshold parameter
- `t + 1` — number of parties required to produce a signature

There are two major operations:

- **DKG / Keygen** — creates the distributed key shares
- **DSG / Signing** — allows a quorum of parties to cooperate and generate a valid ECDSA signature

This is useful for the SocIoTy model because not every device needs to participate in every signing operation.

---

## Implementations Evaluated

### DKLS23

DKLS23 is a Rust implementation of a modern threshold ECDSA protocol.

My work included:

- local benchmark workflows
- DKG benchmarking
- distributed signing benchmarking
- repeated signing / multisign experiments
- fixed and random signer subsets
- distributed Jetstream VM deployment
- TCP-based communication between protocol parties
- controller-side experiment orchestration
- runtime, memory, message, and artifact collection

My fork:

https://github.com/an-cor/dkls23

---

### Binance / BNB Chain tss-lib

`tss-lib` is a Go implementation of multi-party threshold cryptography based on GG-style threshold ECDSA protocols.

The original local benchmark setup used Go channels to simulate multiple parties inside a single machine.

To make the experiment comparable to the distributed DKLS deployment, I extended the project after the semester ended and built a real multi-VM benchmark environment.

My work included:

- a Go TCP relay for communication between parties
- a networked `tss-lib` party runner
- party registration
- broadcast message routing
- point-to-point message routing
- deterministic party identity handling
- distributed key generation across separate VMs
- distributed signing across separate VMs
- arbitrary threshold signer subsets
- fixed multisign experiments
- random multisign experiments
- RSS memory tracking
- Bash experiment orchestration
- Python CSV and results-processing tools
- automated collection of logs and per-party artifacts
- repeated benchmark trials

My fork:

https://github.com/an-cor/tss-lib

Distributed benchmark branch:

`angel-jetstream-vm-benchmarks`

---

## Distributed Architecture

The cloud experiments were deployed using Jetstream virtual machines.

The general architecture was:

```text
                     Controller VM
                          |
                          |
                    Experiment Scripts
                          |
                          v
                     TCP Relay
                  /       |       \
                 /        |        \
                v         v         v
            Party VM   Party VM   Party VM
               1          2          3
                \         |         /
                 \        |        /
                  \       |       /
                   Threshold ECDSA
                        Protocol
                           |
                           v
                Metrics / Logs / Results
```

A controller VM coordinated the experiments while the cryptographic parties ran on separate VMs.

For larger configurations, the environment supported up to **10 party VMs plus one controller VM**.

The distributed `tss-lib` work converted its local in-process message passing into real communication between machines through the relay.

---

## Benchmark Configurations

The base key generation and signing experiments used:

```text
(n=3,  t=2)
(n=5,  t=2)
(n=5,  t=3)
(n=10, t=5)
(n=10, t=6)
(n=10, t=7)
(n=10, t=8)
```

Repeated signing experiments focused on:

```text
(n=3, t=2)
(n=5, t=3)
```

with:

```text
1 signature
5 signatures
10 signatures
```

Two signer-selection modes were evaluated.

### Fixed

The same `t + 1` parties sign during every round.

### Random

A valid subset of `t + 1` parties is selected for each signing round.

---

## Metrics Collected

The benchmarking infrastructure collected measurements including:

- key generation runtime
- signing runtime
- repeated signing runtime
- controller wall-clock time
- maximum resident memory / RSS
- messages sent
- messages received
- bytes sent
- bytes received
- signer sets
- verification status
- per-party logs
- relay logs
- experiment artifacts
- CSV summaries

These measurements allow the implementations to be evaluated as complete software systems rather than only comparing their cryptographic protocols theoretically.

---

## Main Findings

One of the clearest patterns across the experiments was that **distributed key generation is significantly more expensive than signing**.

This has an important implication for SocIoTy.

Instead of generating a new key every time a user performs an operation, a more practical model is:

```text
Generate key shares infrequently
            |
            v
Keep shares on participating devices
            |
            v
Reuse those shares for many signing operations
```

Repeated signing experiments showed why amortizing the expensive setup phase across many signatures is important.

The original local comparison also showed an implementation tradeoff:

- Binance `tss-lib` produced faster runtime measurements in the local experiments.
- DKLS used substantially less memory in those experiments.

The later cloud work was designed to remove an important limitation of the original comparison by running **both implementations in distributed multi-VM environments**.

---

## Project Timeline

### Spring 2026 — Semester Project

During the Advanced Cybersecurity course I:

- studied SocIoTy and threshold ECDSA
- selected DKLS23 and Binance `tss-lib`
- developed local benchmark workflows
- compared key generation and signing
- implemented repeated signing experiments
- collected runtime, RSS, and communication metrics
- built the DKLS Jetstream VM environment
- developed distributed DKLS orchestration
- wrote the project report
- presented the results

At the end of the semester, the distributed Binance deployment remained unfinished.

### Summer 2026 — Continued Development

After the course ended, I continued working on the project.

The main goal was to close the gap between the two implementations by giving `tss-lib` a distributed benchmark environment comparable to DKLS.

I implemented:

```text
tss-lib
├── TCP relay
├── networked party runner
├── distributed keygen
├── distributed signing
├── fixed multisign
├── random multisign
├── RSS tracking
├── experiment orchestration
└── results / CSV tooling
```

The final environment successfully completed the base keygen/sign matrix, fixed multisign matrix, random multisign matrix, and repeated trials with memory tracking.

---

## Code

The protocol implementations and experiment code are maintained in separate forks.

### DKLS23

https://github.com/an-cor/dkls23

### tss-lib

https://github.com/an-cor/tss-lib

Important `tss-lib` benchmark code includes:

```text
cmd/tssbench/
├── relay/
│   └── main.go
└── party/
    └── main.go

scripts/
├── jetstream/
│   ├── run_tss_keygen.sh
│   ├── run_tss_sign_round.sh
│   ├── run_tss_keygen_sign.sh
│   ├── run_tss_base_matrix.sh
│   ├── run_tss_multisign.sh
│   ├── run_tss_multisign_matrix.sh
│   └── run_tss_rss_repeat_trials.sh
│
└── analysis/
    ├── export_tss_keygen_csv.py
    ├── export_tss_sign_csv.py
    └── add_rss_columns.py
```

---

## Repository Purpose

This repository serves as the **project-level home** for the SocIoTy threshold ECDSA evaluation.

It can contain:

```text
.
├── README.md
├── docs/
│   ├── report/
│   └── presentation/
├── figures/
├── results/
│   └── final-summaries/
└── notes/
```

The actual protocol modifications and distributed benchmark implementations remain in their respective DKLS23 and `tss-lib` forks.

---

## Security Note

Some raw threshold ECDSA benchmark artifacts contain generated private key shares.

Files such as:

```text
keygen_save_party_*.json
```

must **not** be committed to this public repository.

Only sanitized benchmark summaries, figures, logs without secret material, and non-sensitive experiment metadata should be published.

The complete raw experiment archive should remain private.

---

## Limitations

The current distributed results use cloud VMs rather than physical smart-home hardware.

Therefore, the results do **not** establish that either implementation is directly suitable for devices such as:

- Raspberry Pi
- Raspberry Pi Zero
- ESP32
- embedded smart-home devices

The VMs do not reproduce the same CPU, memory, storage, networking, or energy constraints as real IoT hardware.

Power consumption was also not measured in the completed VM experiments.

---

## Future Work

Possible extensions include:

- deployment on Raspberry Pi hardware
- deployment on ESP32-class devices where feasible
- power-consumption measurements
- device churn experiments
- adding and removing devices from a threshold group
- deeper analysis of cryptographic bottlenecks
- mapping runtime and memory costs back to individual protocol components

---

## Academic Context

**Project:** Extending SocIoTy to Threshold ECDSA  
**Course:** EE-59903 Advanced Cybersecurity  
**Semester:** Spring 2026  
**Continued development:** Summer 2026

The project builds on:

> Tushar M. Jois, Gabrielle Beck, Sofia Belikovetsky, Joseph Carrigan, Alishah Chator, Logan Kostick, Maximilian Zinkus, Gabriel Kaptchuk, and Aviel D. Rubin. *SocIoTy: Practical Cryptography in Smart Home Contexts.* Proceedings on Privacy Enhancing Technologies, 2024.

---

## Status

**Distributed VM benchmarking complete.**

Both DKLS23 and Binance `tss-lib` have been evaluated using local benchmark workflows, and distributed Jetstream infrastructure was developed for both implementations.

The next stage of the research would be testing on physical IoT hardware and measuring constraints that cloud VMs cannot reproduce.