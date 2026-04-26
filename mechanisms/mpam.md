# ARM MPAM — Memory System Resource Partitioning and Monitoring

> Temporal isolation for mixed-criticality systems. MPAM extends the Identity × Resource → Policy framework from address-space resources to time-domain resources: cache ways and DRAM bandwidth.

**Status:** Introduced in ARMv8.4-A. This page covers architecture, configuration, and safety certification relevance.

---

## Framework Mapping

| Dimension | Value |
|-----------|-------|
| **Identity signal** | PARTID (Partition ID) — carried on every memory transaction |
| **Resource type** | Cache capacity (ways/sets), DRAM bandwidth, interconnect bandwidth |
| **Enforcement side** | Target — at cache controller and DRAM controller |
| **Policy authority** | Hypervisor (EL2) or EL3 firmware |
| **Failure mode** | Bandwidth cap enforcement (throttling, not hard fault); cache eviction restricted to own partition |

---

## Why MPAM Exists — The Temporal Interference Problem

Spatial isolation (SMMU, MPC, TZASC) prevents a non-safety component from *writing* to safety-critical memory. But it cannot prevent:

- A GPU running matrix multiply from evicting the safety controller's cache lines
- An NPU saturating DRAM bandwidth causing safety controller memory accesses to stall
- Non-deterministic cache behaviour making WCET analysis impossible

This is **temporal interference** — and it breaks Freedom from Interference (FFI) under IEC 61508 and ISO 26262 just as surely as a memory write would.

MPAM replaces shared, unpartitioned microarchitectural resources with deterministic per-partition allocations.

---

## Architecture

### PARTID — The Identity Signal

Every memory transaction in an MPAM-enabled system carries a PARTID in its transaction attributes. The PARTID is set by:
- Software writing to `MPAM1_EL1` / `MPAM2_EL2` / `MPAM3_EL3` system registers (for CPU transactions)
- Hardware master configuration (for non-CPU masters, similar to MSW)

PARTID space is hierarchical: EL2 controls which PARTIDs are available to EL1 (similar to VMID delegation in SMMU).

### MPAM-Aware Resources

| Resource | MPAM component | What is partitioned |
|----------|---------------|---------------------|
| L2/L3 cache | Cache Portion Partitioning (CPP) | Cache ways allocated per PARTID |
| DRAM controller | Memory Bandwidth Partitioning (MBP) | Max bandwidth per PARTID |
| Interconnect | Interconnect QoS | Priority and bandwidth on NoC |

### Cache Partitioning (CPP)

Each PARTID is allocated a subset of cache ways via a capacity bitmask. A process with PARTID 2 can only use the ways assigned to partition 2 — it cannot evict lines belonging to PARTID 1's ways, regardless of access patterns.

```
Cache ways:  [0][1][2][3][4][5][6][7][8][9][10][11][12][13][14][15]
PARTID 0:    [■][■][■][■][■][■][■][■]                              ← safety RTOS (8 ways)
PARTID 1:                            [■][■][■][■]                  ← Linux (4 ways)
PARTID 2:                                        [■][■][■][■]      ← NPU inference (4 ways)
```

PARTID 0's cache lines cannot be evicted by PARTID 1 or 2 traffic. WCET analysis for PARTID 0 software can now close — cache behaviour is deterministic and bounded.

### DRAM Bandwidth Partitioning (MBP)

Each PARTID is assigned a maximum DRAM bandwidth fraction (typically expressed as a percentage of peak bandwidth or as a token-bucket rate). The DRAM controller enforces this cap:

```
Total DRAM bandwidth: 50 GB/s

PARTID 0 (safety):    guaranteed minimum 10 GB/s, no cap
PARTID 1 (Linux):     maximum 20 GB/s
PARTID 2 (NPU):       maximum 30 GB/s
```

An NPU running at full throughput cannot starve the safety controller's memory accesses below its guaranteed minimum.

---

## Safety Certification Relevance

### Spatial + Temporal = Complete FFI

| FFI Dimension | Mechanism | What it prevents |
|---------------|-----------|-----------------|
| Spatial | SMMU Stage 2 | Non-safety master writing to safety memory |
| Spatial | TZASC / MPC | Non-safety master reading safety memory |
| **Temporal** | **MPAM CPP** | Non-safety master evicting safety cache lines |
| **Temporal** | **MPAM MBP** | Non-safety master starving safety DRAM access |

Without MPAM, the temporal FFI argument cannot be closed under IEC 61508 or ISO 26262. WCET analysis of safety functions remains open — timing is non-deterministic due to shared cache and bandwidth contention.

### Standards Relevance

| Standard | MPAM contribution |
|----------|------------------|
| IEC 61508 Part 3 Annex F | Evidence that lower-SIL component cannot cause timing failure in higher-SIL component |
| ISO 26262-6 | Support for freedom from interference in ASIL decomposition |
| ISO 13849 | Deterministic response time of safety function unaffected by non-safety workload |

---

## Configuration Example

```c
/* Set PARTID for safety RTOS thread — EL1 */
write_sysreg(MPAM1_EL1,
    MPAM1_EL1_PARTID_I(SAFETY_PARTID) |
    MPAM1_EL1_PARTID_D(SAFETY_PARTID));

/* Configure cache way allocation for PARTID 0 (safety) — EL2 */
mpam_write_cpbm(cache_node, SAFETY_PARTID, 0xFF00); /* upper 8 ways */

/* Configure DRAM bandwidth cap for PARTID 1 (Linux) — EL2 */
mpam_write_mbw_max(dram_node, LINUX_PARTID, 40); /* 40% of peak */
```

---

## Platform Support Status

| Platform | MPAM support | Notes |
|----------|-------------|-------|
| ARM Cortex-A710/A715/A720 | Yes | ARMv8.4-A+ |
| ARM Cortex-X2/X3/X4 | Yes | Full CPP + MBP |
| Neoverse N2/V2 | Yes | Server-class full implementation |
| Cortex-A55/A75/A76 | No | Pre-ARMv8.4-A |
| Qualcomm Snapdragon 8 Gen 3 | Partial | Vendor-specific QoS extensions |

---

## Contribution Opportunities

- [ ] MPAM register map — MPAMIDR_EL1, MPAM1_EL1, MPAM2_EL2 field breakdown
- [ ] Linux kernel MPAM driver status and configuration interface
- [ ] Resctrl filesystem interface for MPAM (Linux `arch/arm64/kernel/mpam/`)
- [ ] Interaction with ARM GIC interrupt latency under MPAM bandwidth caps
- [ ] Hypervisor PARTID virtualisation — how EL2 delegates PARTID space to VMs
- [ ] Real-world WCET analysis methodology using MPAM on automotive SoC
- [ ] Comparison with Intel RDT (Resource Director Technology) — x86 equivalent

---

## References

1. ARM IHI0130 — "ARM Memory System Resource Partitioning and Monitoring (MPAM) Architecture Specification"
2. ARM DEN0107 — "MPAM Software Guide"
3. Linux kernel: `arch/arm64/kernel/mpam/`
4. IEC 61508-3:2010, Annex F — Coexistence of safety-related and non-safety-related software
5. ISO 26262-6:2018, Clause 7 — Freedom from interference
