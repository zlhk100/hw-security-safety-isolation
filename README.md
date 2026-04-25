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

The identity signal may be a CPU TrustZone security state, a DMA StreamID, a Domain ID, or a Partition ID. The resource may be a memory range, a peripheral, a cache partition, or DRAM bandwidth. The policy may be allow/deny, per-domain R/W permissions, or a bandwidth cap.

The structure is always the same. Once you see it, every new SoC's proprietary security controller becomes immediately readable — three questions instead of a new manual.

---

## The Most Important Structural Insight

Every mechanism sits on one of two sides of a bus transaction:

| Side | What it does | Examples | Limitation |
|------|-------------|---------|-----------|
| **Initiator-side** | Tags the transaction with an identity | SAU, IDAU, MSW, VMIDMT, SMMU S1 | Only protects against the CPU |
| **Target-side** | Checks the tag and enforces policy | MPC, PPC, TZASC, XPU, SMMU S2, RDC, MPAM | — |

**Critical gap:** DMA engines, GPUs, and NPUs have no TrustZone state machine. They bypass all initiator-side enforcement. Any asset requiring protection against both CPU compromise AND DMA bypass must have target-side protection.

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
└── use-cases/             ← safety+security co-design patterns (growing)
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
- Interactive SMMU nested walk stepper
- Engineer architecture review checklist

---

## Why Security and Safety Together?

Security isolation (preventing unauthorised access) and functional safety isolation (Freedom from Interference, FFI) use the same hardware mechanisms — SMMU, MPAM, MPC, TZASC, RDC — for structurally identical reasons. Yet they are traditionally treated as separate disciplines with separate certification paths.

This is increasingly untenable in:
- **Software-Defined Vehicles (SDV):** AI inference and ADAS on shared SoC with ISO 26262 safety functions
- **Robotics:** NPU/GPU workloads on same SoC as motor safety controllers (IEC 61508)
- **Industrial automation:** Linux-based IIoT gateways sharing silicon with SIL-rated PLCs
- **Medical devices:** Rich OS alongside IEC 62304 safety-critical firmware

A joint architecture that uses SMMU for spatial FFI and MPAM for temporal FFI can simultaneously support both PSA certification (security) and IEC 61508 SIL2/3 (safety) evidence — leveraging the same hardware investment for both certification paths.

---

## Mechanisms Covered — At a Glance

| Mechanism | Vendor/Standard | Identity Signal | Resource | Side |
|-----------|----------------|-----------------|----------|------|
| SAU / IDAU | ARM (Cortex-M) | CPU TZ state | Memory range | Initiator |
| MPC / PPC | ARM / STM32 | HNONSEC | SRAM / Peripheral | Target |
| TGU | ARM (Cortex-M55+) | CPU TZ state | TCM page | Target |
| MSW | ARM / STM32 GTZC | Injected HNONSEC | — | Initiator |
| TZASC / TZC-400 | ARM | HNONSEC on AXI | DRAM range | Target |
| IO-SMMU (S1/S2) | ARM | StreamID | Address space / PA range | Both |
| VMIDMT | Qualcomm | Initiator ID | — | Initiator |
| XPU (MPU/RPU/APU) | Qualcomm | VMID + NS | Memory / Peripheral | Target |
| RDC | NXP i.MX | Domain ID (DID) | Memory + Peripheral | Target |
| CSU | NXP i.MX | Master identity | Peripheral | Target |
| XMPU / XPPU | Xilinx/AMD | AXI Master ID | Memory / Peripheral | Target |
| TMR / TMZ | AMD GPU | PSP domain / ctx key | DRAM region / page | Target |
| MPAM | ARM (ARMv8.4-A+) | PARTID | Cache ways / DRAM BW | Target |
| DACR | ARM (AArch32) | Domain number | Memory page | Initiator |

---

## Wanted: Mechanisms Not Yet Covered

The following are known gaps — contributions especially welcome:

- **RISC-V IOPMP** — IO Physical Memory Protection for RISC-V systems
- **Apple DART** — Device Address Resolution Table (Apple's IO-SMMU equivalent)
- **MediaTek EMI MPU** — Enhanced Memory Interface Memory Protection Unit
- **Texas Instruments Firewall** — TI SoC hardware firewall architecture
- **Intel VT-d / AMD-Vi** — x86 IO-MMU for comparison
- **RISC-V PMP / ePMP** — Physical Memory Protection in RISC-V cores
- **Arm CCA (Realm Management Extension)** — ARMv9 confidential compute

---

## Getting Started with GitHub Pages

To host this yourself:

1. Fork this repository
2. Go to **Settings → Pages**
3. Set source to **main branch / root**
4. Your article is live at `https://<your-username>.github.io/hw-security-safety-isolation/article/`

---

## Citation

If you use this material in research, training, or publications:

```
Lei zhou (zlhk100). "Hardware Security and Safety Isolation: A Unified Framework."
GitHub, 2025. https://github.com/zlhk100/hw-security-safety-isolation
Licensed under CC BY 4.0.
```

---

## Related Reading

- ARM TrustZone Technical Reference Manual
- ARM SMMU Architecture Specification (IHI0070)
- ARM MPAM Architecture Specification (IHI0130)
- JSADEN002 — PSA Certified Level 2 Lightweight Protection Profile
- IEC 61508 — Functional Safety of E/E/PE Safety-related Systems
- ISO 26262 — Road Vehicles Functional Safety
- Qualcomm: "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)
- ARM: "ARM TrustZone Technology for ARMv8-M" (ARM DEN0083)
