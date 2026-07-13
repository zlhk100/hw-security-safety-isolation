# OCP Rack-Scale Confidential AI: Extending Trust Across a Heterogeneous Compute Platform

> A single CPU CVM is not enough for modern AI inference. Patient medical data processed across CPU clusters, GPU clusters, and NIC-accelerated networking traverses PCIe, CXL, and InfiniBand interconnects. Every link in the data path must be authenticated, integrity-protected, and isolated. This use case maps that complete trust chain.

**Status:** Initial content. Completing the OCP Medium series (Parts 1 and 2).
**Domain:** Cloud infrastructure, AI/ML, healthcare, financial services
**Standards:** PCIe TDISP, DMTF SPDM DSP0274, TCG RATS RFC 9334, HIPAA, ISO 27001

---

## The Problem: The CVM Boundary Stops at the CPU

This use case extends the [confidential computing trusted I/O use case](confidential-computing-trusted-io.md) to a realistic distributed AI workload.

A healthcare organisation deploys AI-based medical image analysis on an OCP rack-scale platform. The workload processes HIPAA-regulated patient data. The threat model:

```
Threat 1 — Physical link attacks
  Sniffing or tampering on PCIe, CXL, or InfiniBand links
  Rogue retimers, switches, cable interposers

Threat 2 — Component spoofing
  Rogue or cloned GPU, NIC, or accelerator
  Malicious firmware on legitimate hardware

Threat 3 — Privileged software compromise
  Cloud provider hypervisor, OS, firmware stack
  Multi-tenant workload interference

Threat 4 — DMA attacks
  PCIe devices performing unauthorised memory access
  Cross-VM data leakage via IOMMU misconfiguration
```

Standard CVM (TDX, SEV-SNP, CCA) addresses Threat 3 for the CPU and memory domain. It does not address Threats 1, 2, or 4 for the device fabric. Patient data moving from CPU memory to GPU local HBM to NIC transmit buffers is exposed at every bus crossing unless each crossing has authentication, encryption, and isolation.

---

## The Complete Trust Chain

```
Layer 1 — CPU + memory: CVM isolation
  Intel TDX / AMD SEV-SNP / ARM CCA
  Hypervisor excluded from TCB
  Patient data encrypted in CPU private memory

Layer 2 — PCIe device: SPDM + IDE + TDISP
  GPU and NIC authenticated via SPDM certificate chain
  PCIe TLPs encrypted with AES-GCM via IDE
  IOMMU S2 managed by TSM (not hypervisor)
  GPU DMA to CVM private memory: hypervisor cannot intercept

Layer 3 — CXL coherent memory: CXL IDE + security extensions
  CXL.mem expanded DRAM pool
  CXL IDE protects FLITs (CXL's equivalent of PCIe TLPs)
  CXL.cache coherent access: different security model from DMA

Layer 4 — InfiniBand RDMA: NDR security (open problem)
  Multi-node GPU cluster communication
  Current state: MACsec at link layer, R_Key/L_Key for memory access
  No TDISP equivalent at InfiniBand maturity
```

---

## The HDCP Analogy: Security Pattern Recognition

Part 2 of the OCP Medium series used HDCP (High-bandwidth Digital Content Protection) as an intuition bridge. The structural parallel is precise and worth preserving:

```
HDCP over HDMI              SPDM + IDE over PCIe/CXL
────────────────────────    ──────────────────────────────────
Device cert auth            GET_CERTIFICATE (SPDM cert chain)
Challenge-response          CHALLENGE / CHALLENGE_AUTH
Session key derivation      KEY_EXCHANGE / FINISH
AES-CTR stream cipher       IDE: AES-GCM on TLPs / FLITs
```

Both solve: point-to-point authentication + session key exchange + link-level encryption. The meaningful security upgrade: AES-GCM (authenticated encryption with integrity + replay protection) vs HDCP's AES-CTR (confidentiality only, no MAC). HDCP was broken partly because it lacked message authentication. IDE's AES-GCM makes link tampering detectable, not just confidential.

---

## Layer 2: PCIe Device Trust Chain (SPDM + IDE + TDISP)

**Authentication and measurement: SPDM over CMA**

Before any patient data moves to a GPU, the CVM (acting through TSM) authenticates the GPU:

```
Step 1 — Certificate authentication (SPDM GET_CERTIFICATE)
  GPU presents certificate chain:
    GPU manufacturer root CA → intermediate CA → device cert
    Device cert contains: hardware identity, public key
    Rooted in IEEE 802.1AR IDevID per DICE architecture

Step 2 — Possession proof (SPDM CHALLENGE / CHALLENGE_AUTH)
  TSM sends nonce
  GPU signs with leaf private key
  TSM verifies signature against presented certificate

Step 3 — Firmware measurement (SPDM GET_MEASUREMENTS)
  GPU reports firmware digest for each measured component
  TSM compares against Reference Values (from GPU vendor's RVP)
  Unexpected digest = attestation failure = no TDISP admission

Step 4 — Session establishment (SPDM KEY_EXCHANGE)
  Derives IDE session keys
  IDE activates: all PCIe TLPs between host and GPU encrypted
```

**Link protection: IDE**

IDE (Integrity and Data Encryption) protects PCIe Transaction Layer Packets (TLPs) using AES-256-GCM. The session keys come from SPDM KEY_EXCHANGE. IDE provides:
- Confidentiality: TLP payload encrypted — link sniffing yields ciphertext
- Integrity: GCM authentication tag — tampering is detected
- Replay protection: sequence number in each TLP — replayed packets detected

**IOMMU isolation: TDISP**

After SPDM attestation passes and IDE is active, TDISP binds the GPU's virtual function (TDI) to the CVM:

```
TSM: LOCK_INTERFACE → TDI transitions CONFIG_LOCKED
TSM: programs IOMMU S2 context:
     GPU TDI StreamID → CVM private memory (patient data pages) only
TSM: TDI transitions to RUN

GPU DMA: reaches patient data pages directly
Hypervisor: cannot remap this IOMMU S2 context
IDE: protects the data on the PCIe bus
```

---

## Layer 3: CXL Coherent Memory — A Different Security Model

CXL (Compute Express Link) is built on the PCIe physical layer but introduces qualitatively different semantics. CXL.cache and CXL.mem enable the CPU and GPU/accelerator to share a coherent address space with load/store semantics at cache-line granularity — not DMA I/O.

This changes the security model in an important way:

```
PCIe DMA:
  Device initiates discrete DMA transactions
  IOMMU controls which physical pages the device can reach
  TDISP/SMMU S2 is the enforcement point

CXL.cache / CXL.mem:
  CPU and CXL device share coherent address space
  Byte-granular load/store access, not discrete DMA
  IOMMU-based enforcement model is insufficient
  Requires CXL IDE (protecting FLITs) + CXL security protocol extensions
```

CXL IDE protects FLITs (Flow Control Units — CXL's packet unit) using AES-GCM, analogously to PCIe IDE protecting TLPs. However, the access control model for cache-coherent CXL memory is still evolving. A CXL device with coherent access to CPU memory has different IOMMU interaction than a standard DMA device.

**Current state:** CXL 3.0 adds security extensions. CXL IDE is specified. The complete security architecture for confidential computing workloads using CXL.cache and CXL.mem coherent access to CVM private memory is an active area of standards development. This is a known open problem in the library — see Wanted list.

---

## Layer 4: InfiniBand RDMA — The Open Problem

For distributed multi-node AI training (multiple GPU servers communicating via InfiniBand), the security architecture is fundamentally different from PCIe/CXL and less mature.

**InfiniBand access control: R_Key and L_Key**

InfiniBand uses memory keys for access control:
- **R_Key (Remote Key):** authorises remote endpoints to DMA into a registered memory region
- **L_Key (Local Key):** authorises local DMA operations

A remote GPU server issues RDMA READ/WRITE targeting memory protected by an R_Key. The InfiniBand HCA checks whether the R_Key matches the target region before forwarding the operation. If an attacker obtains a valid R_Key, they can read or write the corresponding memory region.

**The gap relative to PCIe TDISP:**

| Property | PCIe TDISP | InfiniBand NDR |
|---|---|---|
| Device attestation | SPDM certificate + firmware measurement | No equivalent standard |
| Link encryption | IDE (AES-GCM) | MACsec (AES-GCM at Ethernet/IB layer) |
| Memory access control | IOMMU S2 managed by TSM | R_Key (software-managed by OS) |
| Hypervisor exclusion | TSM programs IOMMU context | Not achieved at standard level |
| Standards body | DMTF + PCI-SIG | InfiniBand Trade Association (IBTA) |

MACsec on NDR InfiniBand provides link-level encryption analogous to IDE — confidentiality and integrity on the wire. But there is no InfiniBand equivalent of TDISP that cryptographically binds device identity to a specific CVM's memory allocation and excludes the hypervisor from remapping R_Keys.

**For the healthcare AI use case:** Multi-node distributed training across InfiniBand cannot currently achieve the same hypervisor-exclusion property as single-node GPU inference via TDISP. This is a real gap in the confidential computing ecosystem.

---

## The Attestation Model for the Complete System

The TCG RATS model (RFC 9334) applies at multiple layers:

```
Platform attestation (CVM ↔ Healthcare org):
  Attester: CVM (TDX TD / SNP VM / CCA Realm)
  Evidence → Verifier (Intel Trust Authority / AMD KDS / ARM Veraison)
  Verifier: consults Endorser (Intel/AMD/ARM cert chain)
            consults RVP (expected TCB measurements)
  AR → Relying Party (healthcare organisation)
  Result: "this CVM is running trusted TCB, verified by hardware attestation"

Device attestation (CVM ↔ GPU via TSM):
  Attester: GPU (SPDM Evidence via CHALLENGE_AUTH + GET_MEASUREMENTS)
  TSM acts as Verifier: checks GPU cert chain (Endorser = GPU mfr)
                        checks GPU firmware digest (RVP = GPU vendor)
  Result: GPU admitted to TDISP RUN state → IOMMU S2 context established

Data flow after attestation:
  Healthcare org → provisions patient data to CVM (TLS over attested channel)
  CVM → TDISP-attested GPU: GPU DMA reaches patient data directly
  GPU → inference computation → results returned via TLS
  At no point: hypervisor, cloud provider staff, or other tenants can access data
```

---

## The Trust Hierarchy Summary

```
Most trusted (hardware-rooted):
  CPU RoT (Intel ME / AMD PSP / ARM RoT)
    ↓ measures
  Platform firmware (BIOS/UEFI)
    ↓ measures
  CVM TCB (TDX module / AMD SP / ARM RoT Monitor)
    ↓ attests to healthcare org via Verifier
  CVM workload (AI inference runtime)
    ↓ verifies GPU via SPDM/TDISP
  GPU (attested hardware + firmware)
    ↓ protected by IDE on PCIe link
  PCIe physical link (IDE)

Not trusted (explicitly excluded from data path):
  Cloud provider hypervisor
  Cloud provider OS and kernel
  Other tenant VMs
  PCIe retimers and switches (for confidentiality: IDE; for identity: TDISP)
```

---

## Contribution Opportunities

- [ ] CXL 3.0 security extensions — complete specification of CXL IDE and coherent memory security
- [ ] InfiniBand NDR security — detailed analysis of R_Key/L_Key model and gaps vs TDISP
- [ ] NVIDIA H100/H200 — Hopper Confidential Computing: GPU TEE + NVLink security
- [ ] AMD Instinct MI300 + SEV-SNP — GPU CVM integration with AMD IOMMU + RMP
- [ ] OCP Security Appraisal Framework (SAF) — how OCP formal security reviews apply
- [ ] Multi-node attestation — how to extend RATS to attest N-node GPU clusters as a unit
- [ ] CXL + TDISP — whether TDISP is being extended to cover CXL fabric devices

---

## References

1. PCIe 6.0 ECN — TDISP and IDE specifications
2. DMTF DSP0274 — SPDM Specification v1.2
3. IETF RFC 9334 — RATS Architecture
4. CXL 3.0 Specification — Security chapter
5. InfiniBand Architecture Specification — NDR security
6. Intel — "Intel TDX Connect Architecture Specification" (2023)
7. Lei Zhou — "Confidential Computing and OCP — Part 1" (Medium, March 2024)
8. Lei Zhou — "Confidential Computing and OCP — Part 2: Interconnect and Security Protocol" (Medium, April 2024)
9. OCP — Open Compute Project Security Specifications (opencompute.org)
