# DMA Security and Trusted I/O

> DMA is the blind spot in every CPU-centric security model. The CPU's identity, privilege level, and TrustZone state are irrelevant the moment a DMA engine generates a bus transaction. This page builds a complete understanding of DMA security from first principles — from the basic IOMMU, through ATS performance optimisations and their attack surfaces, to the TDISP protocol stack that extends cryptographic attestation to PCIe devices in confidential computing environments.

**Status:** Initial content — contributions welcome. See contribution opportunities at the end.
**Standards:** PCIe ATS Specification, PCIe TDISP (ECN to PCIe 6.0), DMTF SPDM DSP0274, TCG TDISP, Intel TDX Connect

---

## The "Aha" Opening: One Bus, No Walls

The CPU enforces its own security. SAU, MPU, privilege levels, TrustZone — all of these are mechanisms the CPU applies to transactions it generates. When you configure TrustZone SAU to mark a memory region Secure, what you are actually saying is: "when the CPU accesses this address in NonSecure state, the CPU will fault."

That protection is completely invisible to a DMA engine.

A DMA engine is a bus master with no TrustZone state machine, no SAU, no privilege register. It places an address on the AXI bus along with whatever HNONSEC value it was configured with, and the transaction proceeds. Your carefully configured SAU regions are not consulted. Your MPU configurations are irrelevant. The DMA engine's transaction travels directly to the SMMU, and if the SMMU has a valid mapping for it, the transaction reaches physical memory.

This is not a bug. It is the fundamental architecture of a shared bus: every master generates transactions independently. The security implication is that every access control mechanism that operates inside the CPU — which is most of them — provides zero protection against DMA.

The IOMMU/SMMU closes this gap for the device path. But it introduces its own attack surface around one concept that does not exist in the CPU world: **mapping lifecycle**.

---

## The Framework Extension: A Fourth Axis

The unified framework across this library is:

```
Identity × Resource → Policy
```

DMA security with ATS reveals that this formula has a hidden assumption baked in: **policy is static**. Configure it, lock it, it holds.

DMA mappings are dynamic. A device is granted a mapping to a physical page, uses it, and then the mapping is revoked. The physical page may be reallocated to a different process or VM. If any hardware component still holds a cached translation to that page when reallocation happens, the result is unauthorised access to another party's memory.

The formula must be extended:

```
Identity × Resource × Lifecycle_State → Policy
```

Lifecycle state captures: is this mapping currently active? Has revocation completed in all hardware caches? Is the physical page safe to reallocate?

Most DMA attacks do not violate the static policy. The IOMMU was correctly configured, the StreamID was correctly assigned, the permissions were correct. The attack exploits the **gap between policy revocation and hardware enforcement** — the window after `dma_unmap()` but before the stale translation has been flushed from every cache that holds it.

Understanding this changes how you think about DMA security: it is not just about getting the access control configuration right. It is about ensuring that configuration changes propagate atomically and completely to every hardware component that caches derived state.

---

## DMA After Unmap: The Hardware Use-After-Free

Software engineers have a name for this pattern. It is called use-after-free.

```
Software Use-After-Free           DMA After Unmap (structural equivalent)
─────────────────────────────     ─────────────────────────────────────────
malloc() → pointer p              dma_map_single() → IOVA mapping to page P
  [P is allocated to this caller]   [device is granted access to page P]

program reads/writes via p        device DMA reads/writes page P via IOVA
  [legitimate use]                  [legitimate use]

free(p)                           dma_unmap_single(IOVA)
  [OS marks P as available]         [IOMMU mapping removed from page table]
  [p is now dangling]               [device ATC still caches IOVA → P]

OS reallocates P to process B     kernel reallocates page P to VM-B

program dereferences p            device DMA using stale cached IOVA
  [accesses process B's data]       [accesses VM-B's physical page]
  ← USE AFTER FREE                  ← DMA AFTER UNMAP
```

The dangling pointer is the stale ATC (Address Translation Cache) entry inside the PCIe device. The page reallocation is the physical page being assigned to a different VM or process. The exploit primitive is identical: one party believes the resource is freed; another party retains a valid hardware handle to it.

The mitigation pattern is also structurally analogous:

| Software memory safety | DMA lifecycle safety |
|---|---|
| ASAN red zones catch dangling pointer access | IOMMU guard regions catch out-of-range IOVA |
| Rust borrow checker: no reference outlives its owner | TSM ordered revocation: no translation outlives its mapping |
| GC defers free until reference count reaches zero | Three-tier shootdown defers page reuse until all caches are cleared |
| Memory-safe languages eliminate the class entirely | Trusted I/O / TSM removes the class from the trusted path |

The software analogy gives embedded and systems engineers immediate intuition: this is the same class of bug you have been fixing in C for years, expressed in hardware DMA translation caches instead of heap allocator metadata.

---

## The Translation Cache Hierarchy: Three Independent Caches

When a DMA mapping is revoked, there are three independent hardware caches that may hold stale translations derived from that mapping. Each requires its own independent invalidation protocol, and they must complete in strict order before the physical page can be safely reused.

```
┌─────────────────────────────────────────────────────────────┐
│ CPU cores                                                    │
│   Core 0: TLB [VA→PA cache]    Core 1..N: TLB [VA→PA cache] │
│   Shootdown: IPI via smp_call_function_many()               │
│   Hardware: INVLPG (x86) / TLBI (ARM)                       │
└────────────────────┬────────────────────────────────────────┘
                     │ CPU generates transactions directly
                     │ (CPU bypasses SMMU — separate path)
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ SMMU / IOMMU                                                 │
│   IOTLB [IOVA→PA cache]                                      │
│   Shootdown: CMD_TLBI written to command queue (MMIO)        │
│   Completion: CMD_SYNC → interrupt or semaphore poll         │
└────────────────────┬────────────────────────────────────────┘
                     │ Device transactions pass through SMMU
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ PCIe devices                                                 │
│   ATC [IOVA→HPA cache] inside the device endpoint           │
│   Shootdown: ATS Invalidation Request → Completion          │
│   Protocol: PCIe link round-trip, device must acknowledge   │
└─────────────────────────────────────────────────────────────┘
```

### The Three Shootdown Protocols

**① ATC Shootdown (PCIe link)**

The SMMU issues command `CMD_ATC_INV`, which causes the root complex to send an ATS Invalidation Request message downstream over the PCIe link to the device endpoint.

The device endpoint must: stop new DMA using the invalidated translation; wait for all outstanding read requests referencing that address to retire; clear the ATC entry; then send an ATS Invalidation Completion message upstream to the root complex.

The host must wait for the Invalidation Completion before proceeding. The PCIe specification requires that the Completion arrive at the root complex *after* any previously posted writes using the invalidated address. This ordering guarantee is what makes the protocol safe — but it requires the device to cooperate. A malicious device can simply not send the Completion.

**② IOTLB Shootdown (MMIO to SMMU)**

The OS writes a `CMD_TLBI_NH_VA` (or appropriate variant) to the SMMU command queue via MMIO. The SMMU processes it in FIFO order and invalidates the relevant IOTLB entry. The OS then writes a `CMD_SYNC` command. The SMMU signals completion of all commands up to the SYNC via interrupt or by updating a semaphore in memory. The OS waits for this signal before proceeding.

Unlike the ATC shootdown, this is entirely within the trusted software stack — no PCIe round-trip, no device cooperation required.

**③ CPU TLB Shootdown (IPI)**

When the kernel unmaps a page from the CPU's page tables (e.g., via `munmap()` or `mprotect()`), it must invalidate TLB entries on all CPU cores that may have cached the now-invalid translation. The kernel calls `smp_call_function_many()` with the `cpu_vm_mask` bitmask, targeting only CPUs running the affected process. Each remote CPU receives an IPI, pauses its current task, and executes a TLB flush (`INVLPG` on x86, `TLBI` on ARM). The initiating CPU waits for acknowledgments from all target CPUs before proceeding.

On ARM, there is also a broadcast TLBI hardware mechanism that eliminates the interrupt-handler overhead by handling the flush in hardware at each core, without interrupting the running task.

**The Safe Revocation Order**

All three must complete before the physical page can be safely reallocated:

```
① ATC shootdown    — PCIe Inv Req → Completion
        ↓
② IOTLB shootdown  — CMD_TLBI → CMD_SYNC completion
        ↓
③ TLB shootdown    — IPI → INVLPG/TLBI → ACK from all cores
        ↓
④ Physical page safe to unmap and reallocate
```

Any reordering creates an attack window. The ATC step is the most expensive (PCIe round-trip, device cooperation required) and the most commonly skipped in practice. The ATC step is also the one a malicious device can intentionally fail to complete — making device attestation a prerequisite for reliable lifecycle enforcement.

---

## ATS: The Performance/Security Tradeoff

Address Translation Services (ATS) allows a PCIe device to request a translation from the IOMMU and cache the result locally in its ATC. This eliminates repeated IOMMU walk latency for the same address, improving DMA throughput significantly for high-bandwidth devices like GPUs and NICs.

But ATS introduces the attack surface described above, and the severity depends critically on which variant of ATS is used.

### Full ATS: The HPA Exposure Problem

In full ATS, the IOMMU returns the final Host Physical Address (HPA) to the device. The device caches it in its ATC. When the device subsequently DMA-accesses that address, it presents the cached HPA directly on the PCIe bus, and the memory controller services it without the IOMMU re-validating the address.

This creates a critical exposure: once the device holds an HPA, the IOMMU is bypassed for all subsequent DMA using that cached address. A device that can modify its own ATC entry (HPA_legit → HPA_forged) can reach arbitrary physical memory.

Full ATS is still required for one specific use case: Peer-to-Peer (P2P) DMA between devices. P2P DMA requires the initiating device to present a physical address on the PCIe fabric that the target device and interconnect can resolve. A guest-physical address (GPA/IPA) cannot be resolved by a receiving device — it needs the actual HPA. Where P2P DMA is not required, full ATS should be avoided.

### Split-Stage ATS: The Mitigation

In split-stage ATS, only the Stage 1 result (IOVA → IPA/GPA) is returned to the device. Stage 2 (IPA → HPA) is never exposed to the device. Every DMA using a split-stage ATS cached translation still executes Stage 2 translation inside the IOMMU. There is no HPA for the device to cache, forge, or present directly.

The tradeoff: split-stage ATS has higher per-DMA latency than full ATS (Stage 2 executes on every access), but eliminates the HPA exposure attack surface.

### PRI: Demand-Paging for Devices

The Page Request Interface (PRI) complements ATS by allowing a device to request page fault resolution when a required translation is missing or inaccessible. This enables SVA/SVM (Shared Virtual Memory) where device DMA targets process virtual addresses that may be demand-paged. The device issues a page request, the OS resolves the fault, and the device retries.

PRI reduces the need for long-term physical page pinning — but it shifts the attack surface to the fault-handling infrastructure. Multiple devices or VMs issuing excessive page requests can exhaust IOMMU command, event, and PRI queues, causing a denial-of-service condition for other tenants. This is a temporal interference problem in the safety sense: one domain's misbehaviour affects the responsiveness of another domain's fault handling.

---

## PASID: Extending Identity to Process Level

The standard SMMU identity signal is the StreamID — it identifies the device. But SVA/SVM requires finer granularity: different processes on the same OS should be able to share a device without their memory spaces becoming accessible to each other.

PASID (Process Address Space Identifier) extends the StreamID with a per-process selector. A DMA transaction tagged with (StreamID 0x42, PASID 7) is mapped to process 7's page tables inside the SMMU, independently of a transaction tagged (StreamID 0x42, PASID 12).

This adds a row to the unified framework lookup table:

| Mechanism | Identity Signal | Resource | Side | Authority |
|---|---|---|---|---|
| IOMMU classic | RID (Bus/Device/Function) | IOVA range | Initiator-adjacent | OS kernel |
| IOMMU + SVA/SVM | RID + PASID | Process VA range | Initiator-adjacent | OS + IOMMU driver |
| IOMMU Stage 2 | StreamID / RID | Physical address range | Initiator-adjacent (trusted layer) | Hypervisor EL2 / EL3 |
| Trusted I/O / TSM | Attested device identity | HPA + lifecycle state | Initiator-adjacent | TSM (hardware-rooted) |

PASID spoofing — a device forging another process's PASID — is a real attack class. A device that can present an arbitrary PASID can access any process's virtual address space, subject only to the SMMU's Stage 2 enforcement. TDISP addresses this by moving PASID management into the TSM: PASIDs are issued, bound, and unbound only under TSM authority, not by the OS kernel.

---

## TDISP: From Positional Trust to Attested Trust

Every DMA mechanism described so far uses **positional trust**: the device is trusted because it is at StreamID 0x42 on the PCIe bus. The SMMU maps StreamID 0x42 to a specific translation context, and that context defines what the device can reach. If a malicious device presents StreamID 0x42 — by spoofing its B/D/F (Bus/Device/Function number) — it inherits that trust. If device firmware is compromised after the IOMMU was configured, the compromised firmware operates within the same IOMMU permissions as the legitimate firmware.

**TDISP (TEE Device Interface Security Protocol)** replaces positional trust with **attested trust**.

Before any DMA mapping is granted to a device that will access CVM (Confidential VM) private memory, the device must prove three things cryptographically:

```
1. Hardware identity — the device has a certificate chain rooted 
   in the device manufacturer, proving it is genuine hardware
   (via SPDM / CMA authentication)

2. Firmware measurement — the device firmware has been measured 
   and its digest matches a known-good value
   (via SPDM measurement exchange / DICE certificate)

3. Configuration report — the specific device interface (TDI) 
   has reported its MMIO ranges, capabilities, and security 
   configuration in an Interface Report that the CVM has verified
   (via GET_DEVICE_INTERFACE_REPORT)
```

Only after passing all three does the TSM (TEE Security Manager) program the IOMMU/SMMU to grant that TDI (Trusted Device Interface) access to the CVM's private memory.

### The Four-Layer TDISP Stack

```
┌─────────────────────────────────────────────────────────────┐
│ TDISP  (TEE Device Interface Security Protocol)              │
│ Lifecycle management: TDI state machine                      │
│ CVM ↔ device trust establishment                             │
├─────────────────────────────────────────────────────────────┤
│ IDE  (Integrity and Data Encryption)                         │
│ PCIe TLP encryption and integrity protection                 │
│ Prevents link interception and masquerading                  │
├─────────────────────────────────────────────────────────────┤
│ CMA  (Component Measurement and Authentication)              │
│ PCIe binding of SPDM over DOE (Data Object Exchange)        │
├─────────────────────────────────────────────────────────────┤
│ SPDM (Security Protocol and Data Model, DMTF DSP0274)        │
│ Authentication, firmware measurement, session key exchange   │
│ Device identity certificate chains, DICE alias certs        │
└─────────────────────────────────────────────────────────────┘
```

### The TDI State Machine: Lifecycle as a First-Class Concept

TDISP formalises the mapping lifecycle into a standardised state machine:

```
CONFIG_UNLOCKED → CONFIG_LOCKED → RUN → ERROR
      │                 │           │
      │   VMM can        │  TSM owns │  Fault or
      │   configure      │  IOMMU    │  violation
      │   device         │  context  │
      │                 │           │
      └─ LOCK_INTERFACE ─┘           └─ device restart
                                        required
```

The IOMMU/SMMU S2 context is programmed by the TSM, and the TDI is granted access to CVM private memory only in the `RUN` state. Transitioning out of `RUN` — whether to `ERROR` or back to `CONFIG_LOCKED` — triggers the ordered revocation sequence (ATC + IOTLB + TLB shootdown) before the CVM's physical pages can be accessed by any other party. This is the use-after-free mitigation at the protocol level: the state machine enforces that the "free" (revocation) completes before any reuse is possible.

### How TSM Changes the IOMMU Trust Model

In traditional DMA, the hypervisor programs the IOMMU S2 context. The IOMMU trusts the hypervisor. In a confidential computing threat model where the hypervisor itself may be compromised, this is insufficient.

TDISP moves S2 programming authority to the TSM — a smaller, more constrained trusted computing base that is not the hypervisor. The TSM is measured and attested, runs at a privilege level the hypervisor cannot modify, and is the sole entity authorised to transition a TDI to `RUN` state and program its IOMMU context.

From the framework perspective: the **Policy Authority** column for Trusted I/O changes from "Hypervisor EL2" to "TSM (hardware-rooted, measured)." This is the architectural significance of Trusted I/O — not a new hardware mechanism, but a new trust root for configuring existing hardware mechanisms.

### Current Limitations (Accurate as of Late 2025)

TDISP currently supports only VM-based TEEs (Intel TDX, AMD SEV-SNP, ARM CCA Realm). The OS, device drivers, and other user-mode components within the VM remain in the TCB — TDISP does not protect against a compromised guest OS.

Linux kernel PCI/TSM infrastructure for TDISP is actively being developed. Each platform architecture has distinct TSM API implementations, and each device's DSM (Device Security Manager) has idiosyncratic behaviours around state transitions — making driver development complex.

Post-attestation firmware compromise is not addressed: TDISP attests the device at bind time, not continuously. A device whose firmware is compromised after attestation retains its IOMMU permissions until the next revocation cycle.

---

## What Trusted I/O Does Not Fix

Honest accounting of the remaining attack surface after TDISP:

| Threat | Status | Reason |
|---|---|---|
| DMA after unmap (stale ATC) | Mitigated by TSM ordered revocation | TSM owns the revocation sequence |
| RID / StreamID spoofing | Mitigated by SPDM attestation | Device identity cryptographically bound |
| PASID spoofing | Mitigated by TSM PASID management | TSM controls issuance and binding |
| IOMMU misconfiguration | Mitigated: TSM programs IOMMU, not hypervisor | Smaller, measured TCB |
| Physical PCIe link interception | Mitigated by IDE | TLPs encrypted with session keys |
| Post-attestation firmware compromise | **Not mitigated** | TDISP attests at bind time only |
| Sub-page granularity leaks | **Not mitigated** | 4KB page granularity, shared data within page |
| Rowhammer / physical DRAM attacks | **Not mitigated** | Physical DRAM physics, not a translation issue |
| Guest OS compromise | **Not mitigated (VM-based TEE only)** | OS remains in TCB |
| Device firmware compromise before attestation | Attested but dependent on correct measurement | Attestation quality depends on implementation |

---

## Putting It All Together: The Evolution in Three Generations

```
Generation 1 — Kernel-managed DMA (implicit device trust)
  Device → IOVA → PA
  Trust model: the kernel is correct and the device is legitimate
  Attack surface: kernel bugs, device firmware bugs, physical access

Generation 2 — IOMMU-protected DMA (positional device trust)
  Device → SMMU [StreamID → IOVA → PA] → memory
  Trust model: the device at StreamID X is legitimate, IOMMU is correctly configured
  Attack surface: ATS stale translations, PASID spoofing, RID spoofing,
                  IOMMU misconfiguration, DMA-after-unmap, physical link attacks

Generation 3 — Trusted I/O (attested device trust)
  Device [attested] → SMMU [TSM-managed S2] → CVM private memory
  Trust model: device identity and firmware are cryptographically verified,
               TSM manages IOMMU context, IDE protects the link
  Attack surface: post-attestation firmware compromise, sub-page leaks,
                  Rowhammer, guest OS (for VM-based TEEs)
```

Each generation closes the attack classes opened by the previous one. Generation 3 does not eliminate all DMA attacks — but it removes the hypervisor from the trust path and grounds device identity in hardware-rooted attestation rather than bus position. For confidential computing workloads where the cloud provider infrastructure is in the threat model, this shift is foundational.

---

## Contribution Opportunities

- [ ] RISC-V IOPMP — how does RISC-V address the same DMA gap without an SMMU?
- [ ] AMD SEV-SNP + IOMMU — how SNP's RMP (Reverse Map Table) interacts with IOMMU for CVM page protection
- [ ] Intel TDX Connect — TSM implementation details for TDX, DOE channel mechanics
- [ ] ARM CCA + GPC — Granule Protection Check inside SMMU for Realm isolation (ACAI paper, arXiv:2305.15986)
- [ ] Linux kernel: `drivers/vfio/`, `drivers/iommu/` — how the kernel manages DMA mapping lifecycle today
- [ ] Linux PCI/TSM patch series — current status of TDISP kernel support (LWN Oct 2025)
- [ ] CXL.io and TDISP — does TDISP extend to CXL memory expanders?
- [ ] Rowhammer via DMA — how precisely does DMA-based Rowhammer work and why is it orthogonal to IOMMU?

---

## References

1. ARM IHI0070 — "ARM System Memory Management Unit Architecture Specification" (normative)
2. PCIe Base Specification — Address Translation Services (ATS) and PRI chapters
3. PCIe 6.0 ECN — TEE Device Interface Security Protocol (TDISP)
4. DMTF DSP0274 — "Security Protocol and Data Model (SPDM) Specification"
5. Intel — "Intel TDX Connect Architecture Specification" (2023)
6. Intel — "Software Enabling for Intel TDX in Support of TEE-I/O" (2023)
7. Microsoft Research — "Confidential Computing within an AI Accelerator" (arXiv:2305.15986 methodology context)
8. Synopsys — "How PCIe's TDISP Architecture Improves Interface Security" (2023)
9. LWN.net — "PCI/TSM: Core infrastructure for PCI device security (TDISP)" (October 2025)
10. Linux kernel `Documentation/core-api/cachetlb.rst` — TLB flush interfaces
11. Linux kernel `arch/x86/mm/tlb.c` — IPI-based TLB shootdown implementation
12. linuxvox.com — "Who Performs TLB Shootdown?" (verified Linux shootdown mechanism)
13. semiengineering.com — "PCIe 6.0 ATS: Verification Challenges and Strategies" (ATS invalidation protocol)
14. N. Sharma — "Rethinking DMA Security: Why Trusted I/O Is Foundational for Confidential Computing," LinkedIn (2025) [recommended reading]
