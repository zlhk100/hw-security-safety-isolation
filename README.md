# Hardware Security & Safety Isolation — A Unified Framework

> **One pattern behind every mechanism:** SAU, IO-SMMU, MPU, MPAM, Qualcomm VMIDMT/XPU, NXP RDC, AMD SEV-SNP RMP, ARM CCA GPT/GPC — all instances of the same three-part formula.

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-blue.svg)](LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## The Core Idea

Every hardware protection mechanism in a modern SoC — whether motivated by security, functional safety, or confidential computing — answers the same three-part question:

```
Initiator Identity  ×  Target Resource  →  Access Policy
```

The identity signal may be a CPU TrustZone security state, a DMA StreamID, a Domain ID, a Partition ID, a PASID, or a cryptographically attested device certificate. The resource may be a memory range, a peripheral, a cache partition, DRAM bandwidth, or a physical page. The policy may be allow/deny, per-domain R/W permissions, a bandwidth cap, or lifecycle-gated access.

The structure is always the same. Once you see it, every new SoC's proprietary security controller becomes immediately readable — three questions instead of a new manual.

**Two extensions to the base formula:**

DMA security adds a **fourth axis — Lifecycle State** — because DMA mappings are dynamic. The attack surface is not just who can access what, but whether revocation has completed atomically across every hardware cache before a physical page is reused. See `mechanisms/iommu-dma-security.md`.

Confidential computing refines the **identity signal** — from positional identity (StreamID, bus position) to attested identity (cryptographic proof of hardware and firmware integrity). The same target-side page ownership pattern (MPC → TZASC → RMP → GPT/GPC) scales from binary Secure/NonSecure to per-VM or per-enclave granularity at 4KB. See `platforms/`.

---

## The Most Important Structural Insight

Every mechanism sits on one of two sides of a bus transaction:

| Side | What it does | Examples | Limitation |
|------|-------------|---------|-----------|
| **Initiator-side (CPU class)** | Tags CPU transactions with identity | SAU, IDAU, MPU | CPU transactions only — DMA bypasses entirely |
| **Initiator-side (device class)** | Translates and checks device transactions | IO-SMMU, VMIDMT, MSW | Device path only — CPU bypasses SMMU |
| **Target-side** | Checks every transaction at the resource | MPC, PPC, TZASC, XPU, RDC, XMPU, RMP, GPT/GPC | — |

**The critical architectural point:** IO-SMMU and SAU/MPU are both initiator-side mechanisms — they differ only in which *class* of initiator they govern. IO-SMMU is not a target-side enforcer. It sits between the DMA device and the system interconnect, upstream of the memory and peripheral resources.

The true target-side enforcers sit at the resource and check every transaction regardless of which initiator generated it. Confidential computing page ownership tables (AMD RMP, ARM GPT/GPC, Intel EPCM) are target-side mechanisms at 4KB granularity — the same pattern as MPC and TZASC, applied at finer granularity with richer identity.

---

## What's in This Repository

```
hw-security-safety-isolation/
├── README.md                          ← you are here
├── LICENSE                            ← CC BY 4.0
├── CONTRIBUTING.md                    ← contribution policy and templates
├── article/
│   └── index.html                     ← full interactive article (GitHub Pages)
├── arm-cca-kvm-deep-dive.html         ← ARM CCA + KVM visual architecture reference
├── mechanisms/                        ← individual mechanism deep-dives
│   ├── _template.md
│   ├── smmu.md                        ← IO-SMMU: initiator-adjacent, S1/S2, ATC shootdown
│   ├── mpam.md                        ← ARM MPAM: temporal isolation, PARTID, cache/BW
│   ├── qualcomm-vmidmt-xpu.md         ← Qualcomm VMIDMT + XPU + multi-ROT trust
│   └── iommu-dma-security.md          ← DMA lifecycle, use-after-free analogy, TDISP stack
├── platforms/                         ← per-platform security architecture analyses
│   ├── _template.md
│   ├── snapdragon.md                  ← Qualcomm Snapdragon access control
│   ├── amd-sev-snp.md                 ← AMD SEV-SNP: encryption + RMP two-layer model
│   ├── arm-cca.md                     ← ARM CCA: four worlds, GPT/GPC, KVM/RMM split
│   ├── intel-sgx.md                   ← Intel SGX: EPCM, NSC/SG parallel, TDX comparison
│   └── android-avf.md                 ← Android AVF: pKVM, pvmfw, DICE, split-app model
├── use-cases/                         ← safety+security co-design patterns
│   ├── _template.md
│   ├── sdv-mixed-criticality.md       ← SDV: AI inference + ADAS on shared SoC (SMMU+MPAM)
│   ├── confidential-computing-trusted-io.md  ← TDISP: attested device identity, TSM, RATS
│   └── ocp-rack-scale-confidential-ai.md     ← OCP: rack-scale AI, PCIe/CXL/IB trust chain
└── diagrams/                          ← SVG source files
    └── README.md
```

### The Main Article

**[→ Read the interactive article](https://zlhk100.github.io/hw-security-safety-isolation/article/)**

Covers:
- TrustZone SAU / IDAU / NSC (Cortex-M)
- MPC / PPC / TGU / MSW / GTZC (Cortex-M bus fabric)
- MMU / Exception Levels / DACR (Cortex-A)
- TZASC / TZPC (DRAM and peripheral protection)
- IO-SMMU 2-stage translation — hypervisor and hypervisorless AMP
- **Qualcomm Snapdragon** VMIDMT / XPU / multi-ROT trust hierarchy
- **NXP i.MX8x** RDC / CSU
- **Xilinx/AMD UltraScale+** XMPU / XPPU
- **AMD GPU** TMR / TMZ
- **ARM MPAM** — temporal isolation for mixed-criticality FFI
- Unified lookup table mapping all mechanisms to the framework
- Interactive IO-SMMU nested walk stepper
- Engineer architecture review checklist

---

## Why Security, Safety, and Confidential Computing Together?

These three domains are converging on the same hardware mechanisms for structurally identical reasons:

**Security isolation** prevents unauthorised access — MPC, SMMU, TZASC enforce who can reach what.

**Functional safety (FFI)** prevents unintended interference — the same MPC, SMMU, and MPAM enforce spatial and temporal separation between mixed-criticality components.

**Confidential computing** removes the hypervisor from the trust path — RMP, GPT/GPC, EPCM, and TDISP extend the same target-side page ownership pattern to protect CVM memory and device DMA from privileged infrastructure software.

This is increasingly critical in:
- **Software-Defined Vehicles (SDV):** AI inference + ADAS on shared SoC; ISO 26262 FFI via SMMU+MPAM
- **Robotics and industrial automation:** NPU/GPU alongside SIL-rated safety controllers; IEC 61508 FFI
- **OCP rack-scale AI:** GPU clusters processing regulated data (HIPAA, GDPR) via SPDM+IDE+TDISP
- **Mobile devices:** Android AVF pVMs protecting sensitive computation from compromised host OS
- **Cloud confidential computing:** TDX, SEV-SNP, CCA removing cloud provider from tenant trust path

---

## Mechanisms Covered — At a Glance

### Hardware Isolation Mechanisms

| Mechanism | Vendor/Standard | Identity Signal | Resource | Side |
|-----------|----------------|-----------------|----------|------|
| SAU / IDAU | ARM (Cortex-M) | CPU TZ state | Memory range | Initiator (CPU class) |
| MPC / PPC | ARM / STM32 | HNONSEC | SRAM / Peripheral | Target |
| TGU | ARM (Cortex-M55+) | CPU TZ state | TCM page | Target |
| MSW | ARM / STM32 GTZC | Injected HNONSEC | — | Initiator (device class) |
| TZASC / TZC-400 | ARM | HNONSEC on AXI | DRAM range | Target |
| IO-SMMU S1 | ARM | StreamID → VA | Address space | Initiator-adjacent (untrusted) |
| IO-SMMU S2 | ARM | StreamID → IPA | Physical address range | Initiator-adjacent (trusted layer) |
| VMIDMT | Qualcomm | Initiator ID → VMID | — | Initiator (device class) |
| XPU (MPU/RPU/APU) | Qualcomm | VMID + NS | Memory / Peripheral | Target |
| RDC | NXP i.MX | Domain ID (DID) | Memory + Peripheral | Target |
| CSU | NXP i.MX | Master identity | Peripheral | Target |
| XMPU / XPPU | Xilinx/AMD | AXI Master ID | Memory / Peripheral | Target |
| TMR / TMZ | AMD GPU | PSP domain / ctx key | DRAM region / page | Target |
| MPAM | ARM (ARMv8.4-A+) | PARTID | Cache ways / DRAM BW | Target |
| DACR | ARM (AArch32) | Domain number | Memory page | Initiator (CPU class) |
| IOMMU + PASID (SVA) | ARM / x86 | RID + PASID | Process VA range | Initiator-adjacent |

### Confidential Computing Mechanisms

| Mechanism | Platform | Identity | Resource | Side | Authority |
|-----------|---------|----------|----------|------|-----------|
| RMP (Reverse Map Table) | AMD SEV-SNP | ASID + expected GPA | Physical 4KB page | Target | AMD SP (PSP) |
| GPT / GPC | ARM CCA | Security world (4 worlds) | Physical 4KB granule | Target | Root Monitor (EL3) |
| EPCM | Intel SGX | Enclave instance (SECS) | EPC page (4KB) | Target | CPU enclave mode |
| pKVM Stage-2 | Android AVF | pVM identity | pVM physical pages | Target | pKVM (EL2) |
| Trusted I/O / TSM | PCIe TDISP | Attested device identity | HPA + lifecycle state | Initiator-adjacent | TSM (hardware-rooted) |

---

## Platforms Covered

| File | Platform | Key content |
|------|---------|-------------|
| `platforms/snapdragon.md` | Qualcomm Snapdragon | VMIDMT, XPU, SMMU, multi-ROT trust hierarchy |
| `platforms/amd-sev-snp.md` | AMD EPYC (Milan+) | Two-layer model: per-VM encryption + RMP integrity; PSP authority; VMPL 0–3 |
| `platforms/arm-cca.md` | ARM CCA (Neoverse V3, Cortex-A720) | Four-world model; GPT/GPC; KVM/RMM authority split; RATS attestation |
| `platforms/intel-sgx.md` | Intel Xeon Scalable | EPCM page ownership; NSC/SG parallel; SGX deprecation; SGX vs TDX |
| `platforms/android-avf.md` | Android (Pixel 6+) | pKVM EL2 isolation; crosVM userspace (untrusted); pvmfw; DICE; split-app Binder |

---

## Use Cases Covered

| File | Domain | Key content |
|------|--------|-------------|
| `use-cases/sdv-mixed-criticality.md` | Automotive SDV | SMMU spatial FFI + MPAM temporal FFI; ISO 26262 / IEC 61508 evidence |
| `use-cases/confidential-computing-trusted-io.md` | Cloud / AI accelerators | TDISP protocol stack; TSM as Verifier; RATS three-role model; TDI lifecycle |
| `use-cases/ocp-rack-scale-confidential-ai.md` | OCP rack-scale AI | HIPAA healthcare use case; SPDM+IDE+TDISP end-to-end; HDCP analogy; CXL/IB gaps |

---

## Wanted: Mechanisms and Platforms Not Yet Covered

Contributions especially welcome:

**Mechanisms:**
- **RISC-V IOPMP** — IO Physical Memory Protection for RISC-V systems
- **RISC-V PMP / ePMP** — Physical Memory Protection in RISC-V CPU cores
- **Apple DART** — Device Address Resolution Table (Apple's IO-SMMU equivalent)
- **MediaTek EMI MPU** — Enhanced Memory Interface Memory Protection Unit
- **Texas Instruments Firewall** — TI SoC hardware firewall architecture
- **Intel VT-d** — x86 IOMMU; ATS/PRI implementation; VT-d + TDX interaction
- **CXL 3.0 security extensions** — CXL IDE for FLITs; coherent memory security model vs DMA
- **InfiniBand RDMA security** — R_Key/L_Key model; gaps vs TDISP; NDR security

**Platforms:**
- **Intel TDX** — Trust Domain Extensions; SEPT (Secure EPT) as RMP equivalent; TDX Connect
- **NVIDIA H100/H200 Confidential Computing** — GPU TEE; NVLink security
- **AMD SEV-SNP + AMD IOMMU** — RMP + IOMMU interaction for CVM DMA protection detail

---

## Getting Started with GitHub Pages

To host this yourself:

1. Fork this repository
2. Go to **Settings → Pages**
3. Set source to **Deploy from a branch** → Branch: `main` → Folder: `/ (root)` → Save
4. Your article is live at `https://<your-username>.github.io/hw-security-safety-isolation/article/`

---

## Citation

If you use this material in research, training, or publications:

```
Lei (zlhk100). "Hardware Security and Safety Isolation: A Unified Framework."
GitHub, 2026. https://github.com/zlhk100/hw-security-safety-isolation
Licensed under CC BY 4.0.
```

---

## Related Reading

**ARM architecture:**
- ARM IHI0070 — ARM System Memory Management Unit Architecture Specification
- ARM IHI0130 — ARM MPAM Architecture Specification
- ARM DEN0083 — TrustZone Technology for ARMv8-M
- ARM Den0125 — Arm CCA Software Architecture (April 2025)

**Confidential computing:**
- IETF RFC 9334 — Remote Attestation Procedures (RATS) Architecture
- Intel — "Intel TDX Connect Architecture Specification" (2023)
- AMD — "AMD SEV-SNP: Strengthening VM Isolation with Integrity Protection" (2020)
- AOSP — "Android Virtualization Framework" (source.android.com)

**Security protocols and standards:**
- DMTF DSP0274 — Security Protocol and Data Model (SPDM) Specification
- PCIe 6.0 / TDISP ECN — TEE Device Interface Security Protocol
- LWN.net — "PCI/TSM: Core infrastructure for PCI device security" (October 2025)
- IEEE 802.1AR — Secure Device Identity (DevID)

**Safety:**
- JSADEN002 — PSA Certified Level 2 Lightweight Protection Profile
- IEC 61508 — Functional Safety of E/E/PE Safety-related Systems
- ISO 26262 — Road Vehicles Functional Safety

**Vendor documentation:**
- Qualcomm Technologies — "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)

**Recommended reading:**
- N. Sharma — "Rethinking DMA Security: Why Trusted I/O Is Foundational for Confidential Computing," LinkedIn (2025)
- Lei Zhou — "Confidential Computing and OCP — Part 1 & 2," Medium (2024)
