# Hardware Security & Safety Isolation — A Unified Framework

> **One pattern behind every mechanism:** SAU, IO-SMMU, MPU, MPAM, Qualcomm VMIDMT/XPU, NXP RDC, Xilinx XMPU, AMD TMR — all instances of the same three-part formula.

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-blue.svg)](LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## The Core Idea

Every hardware protection mechanism in a modern SoC — whether motivated by security, functional safety, or both — answers the same three-part question:

```
Initiator Identity  ×  Target Resource  →  Access Policy
```

The identity signal may be a CPU TrustZone security state, a DMA StreamID, a Domain ID, a Partition ID, or a PASID. The resource may be a memory range, a peripheral, a cache partition, or DRAM bandwidth. The policy may be allow/deny, per-domain R/W permissions, or a bandwidth cap.

The structure is always the same. Once you see it, every new SoC's proprietary security controller becomes immediately readable — three questions instead of a new manual.

DMA security extends this formula with a fourth axis — **Lifecycle State** — because DMA mappings are dynamic. The attack surface is not just who can access what, but whether revocation has completed atomically across every hardware cache before a physical page is reused. See `mechanisms/iommu-dma-security.md`.

---

## The Most Important Structural Insight

Every mechanism sits on one of two sides of a bus transaction:

| Side | What it does | Examples | Limitation |
|------|-------------|---------|-----------|
| **Initiator-side (CPU class)** | Tags CPU transactions with identity | SAU, IDAU, MPU | CPU transactions only — DMA bypasses entirely |
| **Initiator-side (device class)** | Translates and checks device transactions | IO-SMMU, VMIDMT, MSW | Device path only — CPU bypasses SMMU |
| **Target-side** | Checks every transaction at the resource | MPC, PPC, TZASC, XPU, RDC, XMPU | — |

**The critical architectural point:** IO-SMMU and SAU/MPU are both initiator-side mechanisms — they differ only in which *class* of initiator they govern. IO-SMMU is not a target-side enforcer. It sits between the DMA device and the system interconnect, upstream of the memory and peripheral resources.

The true target-side enforcers — MPC, PPC, TZASC, XPU, RDC, XMPU — sit at the resource and check every transaction regardless of which initiator generated it. These are the architectural backstops that provide defence-in-depth against any initiator, including those that bypass or misconfigure the initiator-side mechanisms.

---

## What's in This Repository

```
hw-security-safety-isolation/
├── README.md              ← you are here
├── LICENSE                ← CC BY 4.0
├── CONTRIBUTING.md        ← how to contribute
├── article/
│   └── index.html         ← full interactive article (GitHub Pages)
├── mechanisms/            ← individual mechanism deep-dives (growing)
├── platforms/             ← per-SoC analysis (growing)
├── use-cases/             ← safety+security co-design patterns (growing)
└── diagrams/              ← SVG source files (growing)
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

## Why Security and Safety Together?

Security isolation (preventing unauthorised access) and functional safety isolation (Freedom from Interference, FFI) use the same hardware mechanisms — SMMU, MPAM, MPC, TZASC, RDC — for structurally identical reasons. Yet they are traditionally treated as separate disciplines with separate certification paths.

This is increasingly untenable in:
- **Software-Defined Vehicles (SDV):** AI inference and ADAS on shared SoC with ISO 26262 safety functions
- **Robotics:** NPU/GPU workloads on same SoC as motor safety controllers (IEC 61508)
- **Industrial automation:** Linux-based IIoT gateways sharing silicon with SIL-rated PLCs
- **Medical devices:** Rich OS alongside IEC 62304 safety-critical firmware
- **Confidential computing:** Cloud infrastructure where the hypervisor is in the threat model

A joint architecture using SMMU for spatial FFI, MPAM for temporal FFI, and TDISP for attested device identity can simultaneously support PSA certification (security) and IEC 61508 SIL2/3 (safety) — leveraging the same hardware investment for both certification paths.

---

## Mechanisms Covered — At a Glance

| Mechanism | Vendor/Standard | Identity Signal | Resource | Side |
|-----------|----------------|-----------------|----------|------|
| SAU / IDAU | ARM (Cortex-M) | CPU TZ state | Memory range | Initiator (CPU class) |
| MPC / PPC | ARM / STM32 | HNONSEC | SRAM / Peripheral | Target |
| TGU | ARM (Cortex-M55+) | CPU TZ state | TCM page | Target |
| MSW | ARM / STM32 GTZC | Injected HNONSEC | — | Initiator (device class) |
| TZASC / TZC-400 | ARM | HNONSEC on AXI | DRAM range | Target |
| IO-SMMU S1 | ARM | StreamID → VA translation | Address space | Initiator-adjacent (device class, untrusted layer) |
| IO-SMMU S2 | ARM | StreamID → IPA gate | Physical address range | Initiator-adjacent (device class, trusted layer) |
| VMIDMT | Qualcomm | Initiator ID → VMID | — | Initiator (device class) |
| XPU (MPU/RPU/APU) | Qualcomm | VMID + NS | Memory / Peripheral | Target |
| RDC | NXP i.MX | Domain ID (DID) | Memory + Peripheral | Target |
| CSU | NXP i.MX | Master identity | Peripheral | Target |
| XMPU / XPPU | Xilinx/AMD | AXI Master ID | Memory / Peripheral | Target |
| TMR / TMZ | AMD GPU | PSP domain / ctx key | DRAM region / page | Target |
| MPAM | ARM (ARMv8.4-A+) | PARTID | Cache ways / DRAM BW | Target |
| DACR | ARM (AArch32) | Domain number | Memory page | Initiator (CPU class) |
| IOMMU + PASID (SVA) | ARM / x86 | RID + PASID | Process VA range | Initiator-adjacent (device class) |
| Trusted I/O / TSM | PCIe TDISP | Attested device identity | HPA + lifecycle state | Initiator-adjacent (hardware-rooted) |

---

## Wanted: Mechanisms Not Yet Covered

The following are known gaps — contributions especially welcome:

- **RISC-V IOPMP** — IO Physical Memory Protection for RISC-V systems
- **Apple DART** — Device Address Resolution Table (Apple's IO-SMMU equivalent)
- **MediaTek EMI MPU** — Enhanced Memory Interface Memory Protection Unit
- **Texas Instruments Firewall** — TI SoC hardware firewall architecture
- **Intel VT-d** — x86 IOMMU, ATS/PRI implementation, VT-d + TDX interaction
- **RISC-V PMP / ePMP** — Physical Memory Protection in RISC-V cores
- **ARM CCA RME** — Realm Management Extension, GPC inside SMMU for Realm isolation
- **PCIe IDE** — Integrity and Data Encryption, link-level protection detail
- **CXL.io + TDISP** — does TDISP extend to CXL memory expansion devices?
- **AMD SEV-SNP + RMP** — Reverse Map Table interaction with IOMMU for CVM page protection

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
GitHub, 2025. https://github.com/zlhk100/hw-security-safety-isolation
Licensed under CC BY 4.0.
```

---

## Related Reading

- ARM IHI0070 — ARM System Memory Management Unit Architecture Specification
- ARM IHI0130 — ARM MPAM Architecture Specification
- ARM DEN0083 — TrustZone Technology for ARMv8-M
- JSADEN002 — PSA Certified Level 2 Lightweight Protection Profile
- IEC 61508 — Functional Safety of E/E/PE Safety-related Systems
- ISO 26262 — Road Vehicles Functional Safety
- DMTF DSP0274 — Security Protocol and Data Model (SPDM) Specification
- PCIe 6.0 / TDISP ECN — TEE Device Interface Security Protocol
- Intel — "Intel TDX Connect Architecture Specification" (2023)
- LWN.net — "PCI/TSM: Core infrastructure for PCI device security (TDISP)" (October 2025)
- Qualcomm Technologies — "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)
- N. Sharma — "Rethinking DMA Security: Why Trusted I/O Is Foundational for Confidential Computing," LinkedIn (2025)
