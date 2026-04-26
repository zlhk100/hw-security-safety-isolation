# SDV Mixed-Criticality: AI Inference + ADAS on a Shared SoC

> How to use SMMU (spatial FFI) + MPAM (temporal FFI) to co-locate a non-safety AI inference workload with an ISO 26262 ASIL-B/D safety function on the same SoC — and make a defensible joint certification argument.

**Status:** Skeleton — architecture section complete, certification evidence section wanted.
**Domain:** Automotive SDV (Software-Defined Vehicle)
**Standards:** ISO 26262, IEC 62443, PSA Certified

---

## Problem Statement

Modern automotive SoCs (Snapdragon Ride, Nvidia Orin, Renesas R-Car) co-locate:

- **AI inference workloads** (perception, path planning) — QM or ASIL-A, running Linux + GPU/NPU
- **Safety functions** (emergency braking, lane keeping) — ASIL-B or ASIL-D, running an RTOS on a safety core

Both domains share:
- The same DRAM (physical memory on the same bus)
- The same L3 cache and memory controllers
- The same system interconnect (NoC)

The safety argument requires proof of **Freedom from Interference (FFI)**: the AI workload cannot corrupt or delay the safety function, even in failure modes.

Without hardware enforcement, FFI must be argued through software means (stack isolation, AUTOSAR MemMap) — which regulators increasingly reject as insufficient for ASIL-D.

---

## Hardware Architecture

### Spatial Isolation — SMMU

```
AI Domain (Linux + GPU + NPU)              Safety Domain (RTOS + Safety Core)
────────────────────────────────           ──────────────────────────────────
StreamID: GPU=0x10, NPU=0x11               StreamID: Safety_DMA=0x20
SMMU S2 context: AI_domain                SMMU S2 context: Safety_domain
  └─ maps to: AI_RAM (0x80000000–0xBFFF)    └─ maps to: Safety_RAM (0xC0000000–0xCFFF)

Cross-domain DMA attempt:
  GPU (StreamID 0x10) → 0xC0001000 (Safety_RAM)
  SMMU S2 for AI_domain: no mapping for 0xC0001000
  → Translation fault → DMA aborted
```

**Configuration authority:** EL3 firmware (TF-A) programs both S2 contexts during secure boot and locks SMMU configuration before Linux launches.

### Temporal Isolation — MPAM

```
Cache way allocation (16-way L3):
  PARTID 0 (Safety RTOS):    ways 0–7   (8 ways, 50% — guaranteed)
  PARTID 1 (Linux):          ways 8–11  (4 ways, 25% — maximum)
  PARTID 2 (GPU/NPU):        ways 12–15 (4 ways, 25% — maximum)

DRAM bandwidth allocation:
  PARTID 0 (Safety):    minimum guaranteed: 8 GB/s
  PARTID 1 (Linux):     maximum: 16 GB/s
  PARTID 2 (GPU/NPU):   maximum: 24 GB/s
```

**Effect:** Worst-case execution time (WCET) of safety functions is now bounded regardless of AI workload behaviour. WCET analysis can close.

---

## The Joint Safety + Security Argument

### Spatial FFI Claim
> "The AI inference domain (GPU, NPU, Linux DMA) cannot read or write Safety domain memory because the SMMU Stage 2 context for AI StreamIDs contains no mapping for Safety physical addresses, and this configuration is EL3-locked and cannot be modified by any software in the AI domain."

**Hardware evidence:** SMMU Stream Table Entry (STE) configuration, EL3 lock register state.
**Standards mapping:** ISO 26262-6 Clause 7 (FFI), IEC 61508-3 Annex F.

### Temporal FFI Claim
> "The AI inference workload cannot delay safety function execution beyond its WCET bound because MPAM cache way partitioning prevents cache eviction across partition boundaries, and DRAM bandwidth caps prevent AI traffic from consuming safety domain bandwidth allocation."

**Hardware evidence:** MPAM CPBM (cache partition bitmask) configuration, MBW_MAX register values.
**Standards mapping:** ISO 13849 (deterministic response time), ISO 26262-6 Clause 7.

### Security Claim (IEC 62443 / PSA)
> "A remote exploit in the Linux kernel or AI application cannot escalate to physical actuation because SMMU Stage 2 prevents any Linux-domain DMA from reaching safety controller memory or peripheral registers."

---

## Boot Sequence

```
1. EL3 firmware (TF-A) initialises
2. EL3 programs SMMU S2 contexts:
     AI StreamIDs    → AI_domain context (AI_RAM only)
     Safety StreamID → Safety_domain context (Safety_RAM only)
3. EL3 programs MPAM:
     PARTID 0 (Safety): cache ways 0-7, DRAM min 8 GB/s
     PARTID 1 (Linux):  cache ways 8-11, DRAM max 16 GB/s
     PARTID 2 (GPU):    cache ways 12-15, DRAM max 24 GB/s
4. EL3 locks SMMU configuration registers
5. EL3 locks MPAM configuration registers
6. EL3 launches Hypervisor (EL2) or directly launches Safety RTOS + Linux
7. Safety RTOS runs with PARTID 0 — deterministic, bounded
8. Linux + GPU/NPU run with PARTID 1/2 — contained
```

---

## Gaps and Open Questions

- [ ] How does this interact with AUTOSAR Adaptive Platform memory management?
- [ ] Can MPAM PARTID be virtualised for individual containers inside the Linux domain?
- [ ] What is the SMMU fault handling latency — does it affect safety timing?
- [ ] TÜV / SGS-TÜV Saar: has this architecture pattern received any published guidance?
- [ ] Platform-specific: Snapdragon Ride SMMU configuration — public documentation?
- [ ] Platform-specific: Nvidia Orin — how does its proprietary isolation compare?

---

## Contribution Opportunities

- [ ] Add concrete WCET analysis showing how MPAM closes the timing argument
- [ ] Add IEC 62443 zone/conduit mapping for the AI vs safety boundary
- [ ] Add AUTOSAR MemMap interaction analysis
- [ ] Add comparison: hardware FFI vs AUTOSAR-only FFI — what regulators accept

---

## References

1. ISO 26262-6:2018 — Road vehicles functional safety, Part 6: Product development at the software level
2. IEC 61508-3:2010, Annex F — Coexistence of safety-related and non-safety-related software
3. ARM IHI0130 — MPAM Architecture Specification
4. ARM IHI0070 — SMMU Architecture Specification
5. AUTOSAR — Specification of Memory Mapping (AUTOSAR_SWS_MemoryMapping)
