# Confidential Computing: Extending the Trust Boundary to PCIe Devices

> Traditional confidential computing protects the CPU and memory. The moment a GPU or NIC needs to DMA into confidential VM memory, the hypervisor re-enters the trust path — breaking the isolation boundary confidential computing was built to provide. TDISP closes this gap.

**Status:** Initial skeleton — certification evidence section and platform-specific details wanted.
**Domain:** Cloud infrastructure, AI accelerators, confidential computing
**Standards:** PCIe TDISP (ECN to PCIe 6.0), DMTF SPDM DSP0274, Intel TDX Connect, AMD SEV-SNP

---

## The Problem: The GPU Breaks the Confidential VM

Confidential computing (Intel TDX, AMD SEV-SNP, ARM CCA) protects a VM's memory from the hypervisor and other VMs by encrypting it in hardware. The CPU and memory are inside the trust boundary. The hypervisor is outside it.

Then the workload needs a GPU.

To pass data to a GPU for inference or training, the confidential VM must either:

1. Copy data out of its encrypted private memory into a shared (unencrypted) buffer — which the hypervisor can read. The confidentiality guarantee is broken at the GPU interface.
2. Grant the GPU DMA access to its private memory — but IOMMU S2 is programmed by the hypervisor, which is outside the trust boundary. The hypervisor can remap the GPU's IOMMU context to reach the VM's private pages.

Neither option preserves the confidentiality guarantee. This is not a theoretical gap — it is the precise reason GPU-accelerated confidential workloads have historically required data to leave the protected environment at the device boundary.

TDISP is the architectural answer.

---

## The Attack Model: The Hypervisor as Adversary

In a confidential computing deployment, the cloud provider's infrastructure is in the threat model. The hypervisor may be:

- A compromised software stack (remote exploit)
- A malicious cloud operator
- A legitimate hypervisor with a bug that enables privilege escalation

The threat specifically relevant to device DMA:

```
Hypervisor programs IOMMU S2 context for GPU StreamID
  → IOMMU S2 maps GPU DMA to CVM private memory [legitimate setup]

Hypervisor reprograms IOMMU S2 after setup
  → GPU DMA now also reaches hypervisor memory / another VM's memory
  → Hypervisor reads CVM's confidential data via GPU DMA

OR:

Hypervisor never removes its own IOMMU mapping to CVM pages
  → While GPU is running, hypervisor reads in-flight training data
```

The root cause: the hypervisor programs the IOMMU that governs the GPU's access to CVM memory. It is simultaneously the entity being protected against and the entity with configuration authority. This is the same architectural flaw as giving a lock manufacturer the master key.

---

## The TDISP Solution: Moving Configuration Authority

TDISP changes who programs the IOMMU S2 context for the device. Instead of the hypervisor, it is the TSM (TEE Security Manager) — a measured, hardware-rooted component that the hypervisor cannot modify.

```
Traditional DMA to CVM:
  GPU StreamID → IOMMU S2 [programmed by Hypervisor] → CVM private memory
  Trust boundary: Hypervisor must be trusted
  
TDISP-protected DMA to CVM:
  GPU TDI [attested] → IOMMU S2 [programmed by TSM] → CVM private memory
  Trust boundary: Only TSM and CVM hardware must be trusted
  Hypervisor: outside trust boundary
```

The hypervisor loses the ability to remap the GPU's IOMMU context once the TDI is in `RUN` state. The TSM owns that configuration.

---

## The Protocol Flow: How Trust Is Established

The TDISP handshake before any CVM private memory is accessible to the device:

```
Step 1 — SPDM Authentication (device identity)
  VMM calls TSM: establish SPDM session with device
  TSM ↔ Device DSM: SPDM KEY_EXCHANGE, certificate exchange
  Device proves: hardware identity certificate (manufacturer-rooted)
                 firmware measurement (DICE alias cert)
  TSM verifies certificate chain and firmware digest

Step 2 — TDI Interface Report
  TSM sends: GET_DEVICE_INTERFACE_REPORT to device DSM
  Device returns: MMIO ranges, capabilities, security configuration
  CVM verifies: report matches expected configuration

Step 3 — IDE Session Establishment
  TSM ↔ Device: negotiate IDE keys via SPDM
  PCIe TLPs between host and device are now encrypted
  Link interception cannot yield plaintext CVM data

Step 4 — TDI Lock and Assign
  TSM sends: LOCK_INTERFACE → TDI transitions to CONFIG_LOCKED
  TSM programs: IOMMU S2 context for TDI StreamID → CVM private memory
  TSM sends: DEVICE_INTERFACE_REPORT confirmation to CVM
  TDI transitions to RUN state

Step 5 — CVM operates device
  GPU DMA reaches CVM private memory via TSM-managed IOMMU S2
  Hypervisor cannot read or remap this context
  IDE encrypts PCIe TLPs on the link
```

---

## Mapping to the Unified Framework

| Dimension | Traditional DMA | TDISP-Protected DMA |
|---|---|---|
| **Identity signal** | StreamID (positional — bus assignment) | Attested TDI identity (SPDM cert + firmware measurement) |
| **Resource** | CVM private physical memory | Same |
| **Lifecycle state** | Managed by hypervisor (in threat model) | Managed by TSM (outside threat model) |
| **Policy authority** | Hypervisor EL2 | TSM (hardware-rooted, measured) |
| **Link protection** | None — plaintext PCIe TLPs | IDE — encrypted and integrity-protected TLPs |
| **Revocation** | Hypervisor unmap (untrusted) | TSM-ordered: ATC → IOTLB → TLB → unmap |

The fourth axis — Lifecycle State — is the one that confidential computing exposes most sharply. In traditional deployments, lifecycle mistakes cause security vulnerabilities. In confidential computing deployments, the hypervisor itself is the entity that might exploit those lifecycle windows.

---

## What TDISP Does Not Protect Against

Honest accounting:

**Post-attestation firmware compromise:** TDISP attests the device at bind time. If device firmware is updated or compromised after attestation, the TDI retains its IOMMU permissions until explicitly revoked. Continuous attestation is not currently specified.

**Guest OS compromise:** TDISP protects CVM memory from the hypervisor. If the guest OS inside the CVM is compromised, it can still direct the attested GPU to read any memory it maps to the GPU. TDISP does not protect CVM memory from the CVM's own software.

**Sub-page granularity leaks:** IOMMU operates at 4KB page granularity. If a confidential data structure shares a page with non-confidential data, the GPU's page-level access right covers both.

**Side-channel attacks:** Cache timing, power analysis, electromagnetic analysis, and similar attacks are orthogonal to TDISP.

---

## Platform Support Status (as of late 2025)

| Platform | Status | Notes |
|---|---|---|
| Intel TDX Connect | Specification published (2023) | TDX-specific TSM API; Linux PCI/TSM infrastructure in development |
| AMD SEV-SNP | RMP (Reverse Map Table) provides CVM page protection; TDISP integration ongoing | AMD IOMMU interaction with SNP RMP is the key mechanism |
| ARM CCA | GPC (Granule Protection Check) inside SMMU provides Realm isolation | TDISP spec applicability to CCA being explored |
| Linux kernel | PCI/TSM patch series active (LWN Oct 2025) | Per-platform TSM API; device DSM idiosyncrasies require per-device driver work |

---

## Contribution Opportunities

- [ ] Intel TDX Connect flow — detailed TSM API call sequence for TDI assignment
- [ ] AMD SEV-SNP + IOMMU — how RMP interacts with IOMMU S2 for CVM page protection
- [ ] ARM CCA + GPC — Granule Protection Check as the ARM equivalent of TDISP isolation
- [ ] CXL.io — does TDISP cover CXL memory expansion devices and CXL.cache transactions?
- [ ] Measured boot chain for the device — DICE certificate hierarchy from device ROM to firmware
- [ ] Security analysis: what can a compromised hypervisor still observe post-TDISP?
- [ ] Performance impact: TDISP attestation and IDE overhead on GPU workload throughput

---

## References

1. PCIe 6.0 ECN — TEE Device Interface Security Protocol (TDISP 1.0)
2. Intel — "Intel TDX Connect Architecture Specification" (2023)
3. Intel — "Software Enabling for Intel TDX in Support of TEE-I/O" (2023)
4. DMTF DSP0274 — Security Protocol and Data Model (SPDM) Specification v1.2
5. LWN.net — "PCI/TSM: Core infrastructure for PCI device security (TDISP)" (October 2025)
6. Synopsys — "How PCIe's TDISP Architecture Improves Interface Security" (February 2023)
7. Microsoft Research — "Confidential Computing within an AI Accelerator" (2023)
8. ACAI — "Protecting Accelerator Execution with Arm CCA" (arXiv:2305.15986)
