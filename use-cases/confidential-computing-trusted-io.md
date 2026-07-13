# Confidential Computing: Extending the Trust Boundary to PCIe Devices

> Traditional confidential computing protects the CPU and memory. The moment a GPU or NIC needs to DMA into confidential VM memory, the hypervisor re-enters the trust path — breaking the isolation boundary confidential computing was built to provide. TDISP closes this gap by moving from positional device trust to cryptographically attested device trust.

**Status:** Active — RATS attestation model and TSM role corrected per TCG RATS RFC 9334.
**Domain:** Cloud infrastructure, AI accelerators, confidential computing
**Standards:** PCIe TDISP (ECN to PCIe 6.0), DMTF SPDM DSP0274, TCG RATS RFC 9334, Intel TDX Connect, AMD SEV-SNP

---

## The Problem: The GPU Breaks the Confidential VM

Confidential computing (Intel TDX, AMD SEV-SNP, ARM CCA) protects a VM's memory from the hypervisor. The CPU and memory are inside the trust boundary. The hypervisor is outside it.

Then the workload needs a GPU.

To pass data to a GPU, the confidential VM must either:

1. Copy data out of its encrypted private memory into a shared unencrypted buffer — which the hypervisor can read. Confidentiality is broken at the device boundary.
2. Grant the GPU DMA access to private memory — but IOMMU Stage 2 is programmed by the hypervisor, which is outside the trust boundary. The hypervisor can remap the GPU's IOMMU context to reach the VM's private pages at any time.

Neither option preserves the confidentiality guarantee. TDISP is the architectural answer.

---

## The Attack Model: The Hypervisor as Adversary

In a confidential computing deployment, the cloud provider's infrastructure is in the threat model. The hypervisor may be compromised, malicious, or operated by a party the tenant does not trust.

The specific threat relevant to device DMA:

```
Hypervisor programs IOMMU S2 context for GPU StreamID
  → GPU DMA maps to CVM private memory [legitimate initial setup]

Hypervisor reprograms IOMMU S2 after setup
  → GPU DMA now also reaches hypervisor memory / another VM's memory
  → hypervisor reads CVM's confidential data via GPU DMA
```

The root cause: the hypervisor programs the IOMMU that governs the GPU's access to CVM memory. It is simultaneously the entity being protected against and the entity with configuration authority.

---

## The TDISP Solution: Two Architectural Shifts

**Shift 1 — From positional to attested device identity**

Before TDISP: the device is trusted because it occupies StreamID 0x42 on the PCIe bus. Positional trust. If a malicious device can present the same BDF, it inherits the same permissions.

After TDISP: the device must prove cryptographically that it is genuine hardware running measured firmware, before any IOMMU mapping to CVM private memory is established. Identity is bound to attestation, not bus position.

**Shift 2 — From hypervisor to TSM as IOMMU policy authority**

Before TDISP: hypervisor programs IOMMU S2 context. Hypervisor is outside trust boundary.

After TDISP: TSM (TEE Security Manager) programs IOMMU S2 context. TSM is a measured, hardware-rooted component the hypervisor cannot modify.

```
Traditional:
  GPU StreamID → IOMMU S2 [programmed by Hypervisor] → CVM private memory
  Trust dependency: hypervisor must be trusted

TDISP-protected:
  GPU TDI [attested] → IOMMU S2 [programmed by TSM] → CVM private memory
  Trust dependency: only TSM + CVM hardware
  Hypervisor: cannot remap after TDI enters RUN state
```

---

## The Four-Layer TDISP Stack

```
┌─────────────────────────────────────────────────────────────┐
│ TDISP  (TEE Device Interface Security Protocol)              │
│ TDI lifecycle: CONFIG_UNLOCKED→CONFIG_LOCKED→RUN→ERROR       │
│ Binds attested device interface to CVM                       │
├─────────────────────────────────────────────────────────────┤
│ IDE  (Integrity and Data Encryption)                         │
│ AES-GCM on PCIe TLPs / CXL FLITs                           │
│ Confidentiality, integrity, replay protection on the link    │
├─────────────────────────────────────────────────────────────┤
│ CMA  (Component Measurement and Authentication)              │
│ Binds SPDM to PCIe DOE (Data Object Exchange) channel       │
├─────────────────────────────────────────────────────────────┤
│ SPDM (Security Protocol and Data Model, DMTF DSP0274)        │
│ Device identity certificates, firmware measurement exchange  │
│ Session key derivation for IDE                               │
└─────────────────────────────────────────────────────────────┘
```

---

## The TDI State Machine: Lifecycle as a First-Class Concept

TDISP formalises the device lifecycle into a standardised state machine. The IOMMU S2 context for CVM private memory is established only in RUN state — never before attestation completes.

```
CONFIG_UNLOCKED
  │  hypervisor can configure device (MMIO, capabilities)
  │  VMM calls LOCK_INTERFACE
  ▼
CONFIG_LOCKED
  │  TSM verifies SPDM attestation
  │  TSM establishes IDE session
  │  TSM programs IOMMU S2: TDI StreamID → CVM private memory
  │  TSM confirms Interface Report to CVM
  ▼
RUN
  │  GPU DMA reaches CVM private memory via TSM-managed IOMMU S2
  │  IDE protects PCIe link
  │  Hypervisor cannot remap IOMMU context
  │
  ├─ normal termination → CONFIG_UNLOCKED
  └─ fault / violation → ERROR (device restart required)
```

The lifecycle dimension is the connection to the DMA-after-unmap / use-after-free class of attacks: the state machine enforces that revocation (ATC → IOTLB → TLB shootdown) completes before any CVM pages can be accessed by another party.

---

## The Attestation Flow — TCG RATS Model (RFC 9334)

**Critical architectural point:** The Relying Party (tenant / workload owner) does not verify attestation Evidence directly. Evidence verification is delegated to a Verifier service. The Relying Party receives only an Attestation Result.

The three roles are distinct by design:

```
Endorser                        Reference Value Provider (RVP)
(device manufacturer:           (TSM vendor / SW vendor:
 silicon cert chain,             expected firmware measurements,
 IEEE 802.1AR IDevID)            DICE reference values)
         │                               │
         └──────────────┬────────────────┘
                         ▼
              Verifier (e.g., Intel Trust Authority,
                         ARM Veraison, vendor service)
                         │
                         │  receives Evidence from Attester
                         │  checks cert chain against Endorser
                         │  checks measurements against RVP
                         │  produces: Attestation Result (AR)
                         ▼
              Relying Party (tenant / cloud customer)
                         │
                         │  receives AR only — never raw Evidence
                         │  makes trust decision based on AR claims
                         │  provisions secrets to CVM if AR passes
```

**TSM role in this model:** The TSM acts as the **Verifier** for SPDM device Evidence — it appraises the device's SPDM attestation token against Reference Values (expected firmware digests, certificate chains) and decides whether to admit the TDI to RUN state. The TSM is not the Relying Party; it does not make workload-level trust decisions.

The Relying Party (the healthcare company, the AI workload owner) delegates device Evidence verification to the Verifier service and receives only the Attestation Result for the CVM platform. The CVM itself performs SPDM verification of the device through the TSM.

**Complete attestation sequence:**

```
Phase 1 — Platform attestation (CVM ↔ Relying Party)

  Verifier generates nonce → Relying Party → CVM
  CVM produces Evidence: platform token + realm/TDX token
  Evidence → Verifier
  Verifier consults Endorser (silicon mfr cert chain)
  Verifier consults RVP (expected TCB measurements)
  Verifier produces Attestation Result
  AR → Relying Party
  Relying Party makes trust decision
  Relying Party provisions secrets to CVM over TLS

Phase 2 — Device attestation (CVM ↔ GPU via TSM)

  TSM initiates SPDM session with device DSM via CMA/DOE
  TSM: GET_DIGESTS / GET_CERTIFICATE → device certificate chain
  TSM: CHALLENGE / CHALLENGE_AUTH → proves device holds private key
  TSM: GET_MEASUREMENTS → device firmware digest
  TSM (as Verifier): appraises Evidence against Reference Values
    ✓ Certificate chain roots in trusted manufacturer CA (Endorser)
    ✓ Firmware digest matches expected value (RVP)
    ✓ Device holds leaf private key (CHALLENGE_AUTH)
  TSM: KEY_EXCHANGE → establishes IDE session keys
  TSM: LOCK_INTERFACE → TDI transitions CONFIG_LOCKED
  TSM: programs IOMMU S2 context for TDI StreamID
  TSM: TDI transitions to RUN
  CVM: receives confirmation → GPU DMA to private memory now active
```

---

## Mapping to the Unified Framework

| Dimension | Traditional DMA | TDISP-Protected DMA |
|---|---|---|
| **Identity signal** | StreamID (positional, bus-assigned) | Attested TDI identity (SPDM cert + firmware measurement) |
| **Resource** | CVM private physical memory | Same |
| **Lifecycle state** | Managed by hypervisor (in threat model) | Managed by TSM (outside threat model) |
| **Policy authority** | Hypervisor EL2 | TSM (measured, hardware-rooted) |
| **Link protection** | None — plaintext PCIe TLPs | IDE — AES-GCM encrypted and integrity-protected |
| **Revocation** | Hypervisor unmap (untrusted ordering) | TSM-ordered: ATC → IOTLB → TLB → unmap |

---

## What TDISP Does Not Protect Against

| Threat | Status | Reason |
|---|---|---|
| Post-attestation firmware compromise | **Not mitigated** | TDISP attests at bind time only; continuous attestation not specified |
| Guest OS compromise | **Not mitigated (VM-based TEE)** | Guest OS remains in TCB |
| Sub-page granularity leaks | **Not mitigated** | IOMMU operates at 4KB page granularity |
| Physical DRAM attacks | **Not mitigated** | Orthogonal to IOMMU / link protection |
| Side-channel attacks | **Not mitigated** | Cache timing, power analysis orthogonal |
| DMA after unmap | **Mitigated by TSM ordered revocation** | TSM owns ATC→IOTLB→TLB sequence |
| RID / StreamID spoofing | **Mitigated by SPDM attestation** | Device identity cryptographically bound |
| PASID spoofing | **Mitigated by TSM PASID management** | TSM controls issuance and binding |

---

## Platform Support (late 2025)

| Platform | Status | Notes |
|---|---|---|
| Intel TDX Connect | Specification published (2023) | TDX-specific TSM; Linux PCI/TSM infrastructure in development |
| AMD SEV-SNP | IOMMU + RMP integration; TDISP alignment ongoing | AMD IOMMU RMP checks protect CVM pages from DMA |
| ARM CCA | SMMU GPC extension for Realm isolation | TDISP applicability to CCA being explored |
| Linux kernel | PCI/TSM patch series active (LWN Oct 2025) | Per-platform TSM API; per-device DSM idiosyncrasies |

---

## Contribution Opportunities

- [ ] Intel TDX Connect — detailed TSM API call sequence for TDI assignment to TDX TD
- [ ] AMD SEV-SNP + IOMMU — how RMP interacts with IOMMU for CVM page protection against DMA
- [ ] ARM CCA + SMMU GPC — Granule Protection Check as CCA's device DMA isolation mechanism
- [ ] CXL.io + TDISP — does TDISP extend to CXL memory expansion and CXL.cache transactions?
- [ ] DICE certificate hierarchy — from device ROM to firmware alias cert, 802.1AR IDevID/LDevID
- [ ] Post-attestation integrity — what mechanisms could provide continuous device attestation?
- [ ] Performance impact of IDE overhead on GPU inference throughput

---

## References

1. PCIe 6.0 ECN — TEE Device Interface Security Protocol (TDISP 1.0)
2. DMTF DSP0274 — Security Protocol and Data Model (SPDM) Specification v1.2
3. IETF RFC 9334 — Remote Attestation Procedures (RATS) Architecture
4. Intel — "Intel TDX Connect Architecture Specification" (2023)
5. Intel — "Software Enabling for Intel TDX in Support of TEE-I/O" (2023)
6. LWN.net — "PCI/TSM: Core infrastructure for PCI device security (TDISP)" (October 2025)
7. Synopsys — "How PCIe's TDISP Architecture Improves Interface Security" (2023)
8. TCG DICE Attestation Architecture Specification
9. IEEE 802.1AR — Secure Device Identity (DevID)
