# Qualcomm Snapdragon — Security Architecture Analysis

> Platform-level mapping of Qualcomm Snapdragon access control mechanisms to the unified Identity × Resource → Policy framework.

**Status:** Skeleton — mechanism inventory complete, configuration details wanted.
**Reference silicon:** Snapdragon 8 Gen 2/3, Snapdragon Ride (automotive)
**Primary source:** Qualcomm Technologies, "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)

---

## SoC Overview

Snapdragon SoCs are ARM-based with Qualcomm-proprietary security extensions layered on top of the standard TrustZone architecture. Key characteristics:

- Multiple Cortex-A (performance) + Cortex-X (prime) CPU clusters
- Adreno GPU with dedicated SMMU context banks
- Hexagon DSP — dedicated domain, separate security context
- Modem (separate ROT domain from application processor)
- LPDDR5/5X DRAM with TZASC-equivalent protection
- Multiple DMA engines across multimedia, connectivity, and storage

---

## Mechanism Inventory

| Mechanism | Framework Role | Coverage | Authority |
|-----------|---------------|---------|-----------|
| VMIDMT | Initiator-side identity injector | All non-CPU masters | ROT domain (TZ or Qualcomm ROT) |
| XPU (MPU mode) | Target-side memory enforcer | DRAM regions | Resource group owner |
| XPU (RPU mode) | Target-side peripheral enforcer | Peripheral registers | Resource group owner |
| XPU (APU mode) | Target-side fragmented-range enforcer | Non-contiguous ranges | Resource group owner |
| SMMU | Address translation + access control | GPU, Hexagon DSP, multimedia | TZ (secure), Hypervisor (non-secure) |
| TrustZone | CPU-side security state | CPU transactions | EL3 monitor |
| Qualcomm ROT | Parallel trust root | Modem domain | Qualcomm-specific secure subsystem |

---

## DMA Master Coverage

| Master | SMMU coverage | VMIDMT | Notes |
|--------|--------------|--------|-------|
| Adreno GPU | Yes — dedicated SMMU context banks | Yes | PIL-based firmware auth via qcom_scm |
| Hexagon DSP | Yes — separate SMMU domain | Yes | Separate ROT from application TZ |
| Video codec | Yes | Yes | Firmware loaded via PIL |
| Camera ISP | Yes | Yes | |
| Modem | Qualcomm ROT managed | Yes | Mutually distrusting from TZ |
| WiFi/BT | Yes | Yes | |
| USB | Yes | Yes | |

---

## Trust Hierarchy

```
TrustZone (ARM standard)              Qualcomm ROT
  └─ EL3 Secure Monitor                 └─ Modem security domain
  └─ QSEE / QTEE (Trusted OS)           └─ Independent VMIDMT ownership
  └─ Hypervisor (EL2 / QHEE)            └─ Independent XPU resource groups
       └─ Security Domain 1 (Linux)
       └─ Security Domain 2
       └─ Security Domain 3
```

Key property: TrustZone and Qualcomm ROT are **mutually distrusting**. Neither can access or modify the other's VMIDMT configurations or XPU resource groups.

---

## Lock Sequence

```
1. XBL (eXtensible Bootloader) — EL3
   - Initialises VMIDMT for all masters
   - Programs initial XPU resource groups
   - Configures SMMU secure context banks (TZ-owned)

2. QSEE / QTEE launches (EL1 Secure)
   - Manages Trusted Applications
   - Programs DRAM XPU for TEE carve-out

3. Hypervisor / QHEE launches (EL2)
   - Programs SMMU S2 context banks for non-secure masters
   - Controls GPU/DSP StreamID → context bank mapping

4. Linux launches (EL1 NonSecure)
   - Cannot modify SMMU stream table entries or S2 page tables
   - Cannot access secure XPU resource groups
   - Can configure SMMU S1 context banks for its own use cases (delegated by EL2)
```

---

## Firmware Authentication Flow (Adreno GPU Example)

The Linux Adreno GPU driver firmware loading sequence — a concrete demonstration of the full three-component pattern:

```c
/* drivers/gpu/drm/msm/adreno/adreno_gpu.c */

/* Step 1: Load signed firmware blob to DRAM via OS */
ret = request_firmware(&fw, fwname, drm->dev);

/* Step 2: Call TrustZone to authenticate + lock DRAM region */
ret = qcom_scm_pas_auth_and_reset(pasid);
/* Inside this call, TZ:
   - Programs XPU to grant GPU-only access to firmware region
   - Authenticates firmware signature
   - Releases GPU from reset if auth passes */
```

---

## Gaps — Contribution Wanted

- [ ] XBL register configuration detail — what exactly is locked and when
- [ ] QHEE internals — how does it manage SMMU S2 vs standard KVM
- [ ] SHM Bridge protocol — memory descriptor validation between Linux and QTEE
- [ ] Qualcomm ROT isolation — what prevents TZ from accessing modem memory
- [ ] Snapdragon Ride automotive — ASIL decomposition using VMIDMT/XPU
- [ ] PIL protocol detail — pas_id values, authentication certificate format

---

## Platform Template

Use this as the basis for other platform analysis pages. See `platforms/_template.md`.

---

## References

1. Qualcomm Technologies — "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)
2. Linux kernel: `drivers/gpu/drm/msm/adreno/adreno_gpu.c`
3. Linux kernel: `drivers/firmware/qcom_scm.c`
4. Linux kernel: `drivers/iommu/arm/arm-smmu/arm-smmu.c`
5. Qualcomm developer documentation: https://docs.qualcomm.com
