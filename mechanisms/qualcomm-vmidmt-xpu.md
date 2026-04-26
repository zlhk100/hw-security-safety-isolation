# Qualcomm Snapdragon: VMIDMT and XPU

> Qualcomm's proprietary initiator-side identity injector (VMIDMT) and target-side policy enforcer (XPU) — implementing the universal Identity × Resource → Policy framework with support for multiple mutually-distrusting Root of Trust domains.

**Status:** Covered in main article. This page: detailed architecture, configuration tables, and multi-ROT trust model analysis.

---

## Framework Mapping

### VMIDMT

| Dimension | Value |
|-----------|-------|
| **Identity signal** | VMID (Virtual Master ID) + NS/Secure signal |
| **Resource type** | N/A — identity assignment only (initiator-side) |
| **Enforcement side** | Initiator |
| **Policy authority** | ROT domain (TrustZone or Qualcomm ROT) |
| **Failure mode** | N/A — VMIDMT generates attributes; XPU enforces |

### XPU

| Dimension | Value |
|-----------|-------|
| **Identity signal** | VMID + NS signal (from VMIDMT) |
| **Resource type** | Memory range (MPU mode), peripheral registers (RPU mode), fragmented ranges (APU mode) |
| **Enforcement side** | Target |
| **Policy authority** | Resource group owner (ROT domain) |
| **Failure mode** | Bus error → fault to initiating master |

---

## VMIDMT — Virtual Master ID Mapping Table

### Role

VMIDMT is the initiator-side identity injector. It transforms hardware-fixed initiator identifier signals into security attributes that travel with every transaction on the bus.

**Inputs (hardware-fixed):** Initiator identifier — a signal hardwired in silicon that identifies the physical initiator (e.g., channel number for a shared DMA engine).

**Outputs (software-configured):**
- `VMID` — Virtual Master ID, identifies the security sub-domain
- `NS` signal — Secure (0) or NonSecure (1), maps to TrustZone S/NS

### Multi-ROT Support

VMIDMT supports mutually distrusting ROT domains. Each ROT domain configures VMID generation only for its own domain. No ROT domain can influence another's security attributes — this is enforced in hardware.

```
VMIDMT configuration example (shared DMA, 2 channels):

Channel 0 → VMID: CPU_OS,    NS: NonSecure   [configured by TrustZone]
Channel 1 → VMID: TrustZone, NS: Secure      [configured by TrustZone]

TrustZone cannot delegate NS signal configuration to other domains.
TrustZone can delegate VMID generation for specific channels to trusted domains.
```

### Comparison to ARM MSW

| Feature | ARM MSW | Qualcomm VMIDMT |
|---------|---------|----------------|
| Scope | Single master wrapper | Shared across multiple initiators |
| Identity output | HNONSEC + HPROT | VMID + NS signal |
| Multi-ROT | No (single TZ root) | Yes (mutually distrusting ROT domains) |
| Configuration granularity | Per-master | Per-channel (sub-master) |

---

## XPU — eXternal Protection Unit

### Three Operating Modes

XPU mode is determined at hardware design time — each individual XPU instance operates in exactly one mode.

#### MPU Mode (Memory Protection Unit)
- Resource groups are software-programmable contiguous address ranges
- Start/end addresses must be 4 KB boundary aligned
- Equivalent to: ARM MPC, TZASC watermark mode, NXP RDC memory region

#### RPU Mode (Register Protection Unit)
- Resource groups are fixed contiguous address ranges of equal size
- Typically used for peripheral memory-mapped register banks
- Equivalent to: ARM PPC, TZPC, NXP CSU

#### APU Mode (Address Protection Unit)
- Resource groups can contain fixed fragmented or contiguous ranges of varying sizes
- Most expressive mode — no direct ARM equivalent
- Used where a security domain's resources are non-contiguous in the address map

### Resource Group Ownership Model

Each XPU resource group has exactly one owning ROT domain. Key properties:

1. Only the owner can configure the resource group's access policies
2. The owner can grant R, W, or R/W access to other security domains
3. Resource groups from different ROT domains in the same XPU are independently owned
4. Ownership cannot be transferred at runtime

This is a **capability-based model** on top of the identity-based model — not just "what security domain are you," but "which domains have been explicitly granted access to this specific resource group."

### XPU Configuration Example

```
XPU (MPU mode) protecting 96 KB DRAM region:

Resource Group 0: 0x1000_0000 – 0x1001_0000 (64 KB)
  Owner: TrustZone
  Read:  TrustZone, CPU_OS
  Write: TrustZone only

Resource Group 1: 0x1001_0000 – 0x1001_8000 (32 KB)
  Owner: TrustZone
  Read:  CPU_OS
  Write: CPU_OS
```

---

## The Firmware Authentication Pattern

The most instructive use case — illustrates all three components working in sequence:

```
1. CPU OS loads firmware blob to DRAM
   [CPU OS has full access — no XPU protection yet]

2. CPU OS calls TrustZone via SMC (Secure Monitor Call)

3. TrustZone validates the DRAM address range

4. TrustZone programs XPU resource group:
   owner = TrustZone
   read  = Video_CPU only
   write = nobody
   [CPU OS access to firmware region: revoked]

5. TrustZone authenticates firmware signature

6. Auth pass → TrustZone releases Video CPU from reset
   Auth fail → XPU resource group released, error returned

7. Video CPU executes from authenticated, XPU-protected memory
```

Linux kernel implementation: `drivers/gpu/drm/msm/adreno/adreno_gpu.c`
```c
ret = qcom_scm_pas_auth_and_reset(pasid);
```

This is structurally identical to AMD PSP + TMR, OP-TEE + TZASC carve-out, and every TEE-based multimedia firmware loading pattern.

---

## Multi-ROT Trust Hierarchy

Qualcomm's most architecturally distinctive feature vs. ARM's single-root model:

```
TrustZone ROT ──────────────────── Qualcomm ROT
     │                                   │
     │ manages                           │ manages
     ▼                                   ▼
[Hypervisor (EL2)]              [Qualcomm-specific]
     │                          [security services]
     ├── Security Domain 1
     ├── Security Domain 2
     └── Security Domain 3
```

Neither ROT can access or modify the other's VMIDMT configurations or XPU resource groups. This enables true mutual distrust — relevant for devices where the modem (Qualcomm ROT) and the application OS (TrustZone ROT) must be isolated from each other at the hardware level.

---

## Contribution Opportunities

- [ ] SMMU integration on Snapdragon — how StreamID, SSD, and VMIDMT interact
- [ ] QHEE (Qualcomm Hypervisor Execution Environment) — EL2 SMMU S2 management
- [ ] QTEE vs QSEE naming — version history and architecture differences
- [ ] SHM Bridge — shared memory descriptor validation between Linux and TZ
- [ ] PIL (Peripheral Image Loader) — full kernel driver flow for firmware authentication
- [ ] XPU violation handling — interrupt routing and error recovery
- [ ] Automotive Snapdragon Ride — VMIDMT/XPU in ISO 26262 context

---

## References

1. Qualcomm Technologies, Inc. — "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)
2. Linux kernel: `drivers/firmware/qcom_scm.c` — SCM call interface to TrustZone
3. Linux kernel: `drivers/gpu/drm/msm/adreno/adreno_gpu.c` — PIL firmware loading
4. Linux kernel: `drivers/iommu/arm/arm-smmu/` — Qualcomm SMMU integration
