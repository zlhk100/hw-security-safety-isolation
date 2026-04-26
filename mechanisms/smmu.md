# IO-SMMU — System Memory Management Unit

> Target-side address translation and access control for all non-CPU bus masters. The mechanism that closes the DMA gap that SAU, MMU, and MPU cannot address.

**Status:** Core coverage in main article. This page: detailed register-level and configuration reference.

---

## Framework Mapping

| Dimension | Value |
|-----------|-------|
| **Identity signal** | StreamID (hardware-assigned per master or DMA channel) + SSD (Security State Determination) |
| **Resource type** | Physical address space (all DRAM and peripheral ranges) |
| **Enforcement side** | Both — S1 initiator-adjacent (translation), S2 target-side (permission gate) |
| **Policy authority** | S1: Guest OS (EL1) / S2: Hypervisor (EL2) or EL3 firmware |
| **Failure mode** | Translation fault → bus error → abort to initiating master |

---

## Architecture Overview

The SMMU intercepts transactions from non-CPU bus masters (GPU, NPU, DMA, NIC, etc.) before they reach DRAM or peripherals. It resolves virtual addresses to physical addresses and enforces access permissions — performing the same role as the CPU MMU but for all other system masters.

### SMMUv2 vs SMMUv3

| Feature | SMMUv2 | SMMUv3 |
|---------|--------|--------|
| Master identification | Context Bank + Stream Matching | Stream Table (linear or 2-level) |
| Stage 2 support | Yes | Yes |
| PCIe ATS support | No | Yes |
| Command queue | No | Yes (CMD_QUEUE) |
| Event queue | Interrupt-based | Queue-based (EVT_QUEUE) |
| Secure streams | Via secure context banks | Via secure Stream Table |

---

## Two-Stage Translation

### Why two stages are necessary

One-stage SMMU gives the OS control of page tables — but the OS is the entity you do not fully trust in a virtualised or mixed-criticality system. Two-stage translation separates intent (Stage 1, OS-controlled) from permission (Stage 2, hypervisor/EL3-controlled).

### The nested walk — S2 fires at every S1 level

This is the mechanism that makes Stage 2 genuinely unbypassable:

```
Device issues VA
  │
  └─► SMMU looks up S1 L1 table pointer (an IPA)
        │
        └─► S2 resolves L1 table IPA → PA          [S2 check #1]
              │
              └─► SMMU reads S1 L1 entry, gets L2 pointer (IPA)
                    │
                    └─► S2 resolves L2 table IPA → PA    [S2 check #2]
                          │
                          └─► SMMU reads S1 leaf PTE, gets output IPA
                                │
                                └─► S2 resolves output IPA → real PA  [S2 check #3]
                                      │
                                      ├─► Mapped + permitted → PA delivered
                                      └─► Unmapped or denied → Stage 2 fault
```

A malicious Guest OS cannot escape Stage 2 by pointing S1 tables at safety-critical memory — even the table pointers themselves are IPAs that must resolve through S2.

---

## Hypervisor Deployment

Stage 2 context is per-VM. Each VM's StreamIDs are mapped to a Stage 2 context bank/STE containing that VM's physical address range. Cross-VM DMA is structurally impossible — the StreamID's Stage 2 context has no mapping for another VM's physical memory.

**Configuration ownership:**
- EL3: programs secure stream mappings, cannot be modified by EL2 or EL1
- EL2 (Hypervisor): programs non-secure stream → context bank mappings and Stage 2 page tables
- EL1 (OS): configures Stage 1 context banks only, within what EL2 delegates

---

## Hypervisorless AMP Deployment

When no EL2 hypervisor is present (bare-metal AMP: Linux on Cortex-A + safety RTOS on Cortex-R):

1. Stage 1 is **disabled** for DMA masters (not identity-mapped — actually disabled)
2. EL3 firmware programs Stage 2 directly — each StreamID mapped to its domain's physical range only
3. EL3 locks SMMU configuration registers (`SMMU_s_CONF`) before launching Linux
4. Linux at EL1 cannot reconfigure stream table entries or Stage 2 page tables

---

## Software Configuration Example

```c
/* Qualcomm/Linux kernel: PIL firmware authentication + SMMU domain assignment */
ret = qcom_scm_pas_auth_and_reset(pasid);

/* ARM SMMU driver: stream table entry configuration */
arm_smmu_write_strtab_ent(smmu_domain, sid, &cfg);
```

---

## Lock Mechanism

| Platform | Lock register | Effect |
|----------|--------------|--------|
| Generic ARM SMMU | `SMMU_s_GBCR.SMMUSID_DISABLE` | Prevents NS software modifying secure streams |
| Qualcomm | SSD configuration in TrustZone | TZ owns secure context banks permanently |
| General EL3 | Write-protect SMMU base registers from NS | EL1/EL2 cannot reach configuration registers |

---

## Comparison to Similar Mechanisms

| Mechanism | Scope | Identity | Authority |
|-----------|-------|----------|-----------|
| SAU (Cortex-M) | CPU transactions only | CPU TZ state | Secure FW |
| TZASC / TZC-400 | DRAM range check only | HNONSEC | EL3 |
| **IO-SMMU** | All non-CPU masters, full address space | StreamID | EL2 / EL3 |
| Qualcomm XPU | Per-resource group | VMID | ROT domain |
| NXP RDC | Per-domain memory + peripheral | DID | EL3 |

SMMU is the most comprehensive non-CPU master protection — covering translation, permission, and isolation in one component. TZASC is simpler (range-only, no translation) and serves as a backstop for transactions that reach DRAM regardless of SMMU outcome.

---

## Contribution Opportunities

- [ ] SMMUv3 Stream Table format — linear vs 2-level, STE field breakdown
- [ ] SMMU fault handling — EVT_QUEUE processing, error recovery
- [ ] PCIe ATS (Address Translation Services) interaction with SMMU S1
- [ ] SVA (Shared Virtual Addressing) — SMMU S1 shared with CPU MMU
- [ ] Platform-specific: Qualcomm SMMU context bank ownership model
- [ ] Platform-specific: ARM Mali GPU SMMU integration

---

## References

1. ARM IHI0070 — "ARM System Memory Management Unit Architecture Specification"
2. Linux kernel: `drivers/iommu/arm/arm-smmu-v3/`
3. ARM DEN0125 — "SMMU Software Guide"
4. Qualcomm Technologies: "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)
