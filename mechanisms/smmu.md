# IO-SMMU — System Memory Management Unit

> The IO-SMMU is **not** a target-side gatekeeper. It is an initiator-side translation and access control layer for non-CPU bus masters. Understanding this distinction is the most important architectural insight about the SMMU.

**Status:** Revised — initiator-side classification corrected, DMA lifecycle attack surface added.

---

## The "Aha" Moment Before Anything Else

Most engineers encountering the SMMU think of it as "a firewall in front of memory." That mental model is wrong in a subtle but consequential way.

A firewall sits at the target — at the door of the resource being protected. The SMMU sits at the **initiator** — between the device and the bus. It intercepts the device's transaction *as it leaves the device*, translates the address, checks permissions, and either forwards or blocks the transaction before it ever reaches the interconnect fabric.

The practical consequence: the SMMU does not protect memory from being reached by other paths. A different device without an SMMU on its path, or a device that bypasses its SMMU through a misconfiguration, can still reach the same physical memory. Target-side mechanisms — TZASC, MPC, XPU — are the true gatekeepers at the resource. The SMMU is initiator-side enforcement for device transactions, exactly as the SAU is initiator-side enforcement for CPU transactions.

```
CPU transactions    →  SAU / MPU governs (initiator-side, CPU class)
Device transactions →  SMMU governs     (initiator-side, DMA/device class)

Both are initiator-side. They cover different initiator classes.
Neither is a target-side enforcer.

The target-side enforcers are: TZASC, MPC, PPC, TGU, XPU, RDC.
```

---

## Framework Mapping

| Dimension | Value |
|-----------|-------|
| **Identity signal** | StreamID — hardware-assigned per device or per DMA channel |
| **Resource type** | The device's virtual address space (what the device sees) |
| **Enforcement side** | **Initiator-adjacent** — between device and system interconnect |
| **Policy authority** | S1: Guest OS (EL1) — untrusted. S2: Hypervisor (EL2) or EL3 firmware — trusted |
| **Failure mode** | Translation fault → abort, returned to initiating device |

### S1 vs S2: A Trust Authority Distinction, Not a Bus Position Distinction

Both Stage 1 and Stage 2 run inside the same SMMU hardware, upstream of the interconnect. The difference is not where they sit — it is who controls them:

| Stage | Controlled by | Trust level | What it expresses |
|-------|-------------|-------------|-------------------|
| S1 | Guest OS (EL1) | Untrusted | Device's intended address (VA → IPA) |
| S2 | Hypervisor (EL2) / EL3 | Trusted | Boundary the device is contained within (IPA → PA) |

S2 is the security boundary not because it is target-side, but because **the trusted entity owns it**. This is the same architectural principle as the SAU being controlled by Secure firmware and locked by SSC/HDP — the mechanism's security comes from its control authority, not its bus position.

---

## Why the SMMU Exists — The Gap SAU and MPU Cannot Fill

SAU (Cortex-M) and MPU (Cortex-A) enforce access control based on the CPU's privilege state. When the CPU generates a transaction, the SAU checks whether the CPU's current security state permits this access.

DMA engines, GPUs, NPUs, and NICs generate bus transactions independently. They have no TrustZone state machine. They generate AXI/AHB transactions and present them directly on the bus fabric — with whatever StreamID and address they choose. There is nothing in the CPU's security infrastructure that governs these transactions.

This is the gap: **every non-CPU master is invisible to SAU and MPU.**

The SMMU closes this gap by intercepting device transactions before they reach the interconnect and subjecting them to the same kind of identity-based access control that SAU/MPU provides for CPU transactions.

```
Before SMMU:
  Device → raw AXI transaction → bus fabric → any memory
  CPU    → SAU-checked    → bus fabric → policy-enforced

After SMMU (initiator-side coverage for devices):
  Device → SMMU → translated + permission-checked → bus fabric
  CPU    → SAU  → checked → bus fabric
```

The bus fabric and target-side mechanisms (TZASC, MPC) still provide their own enforcement — they are independent layers. The SMMU does not replace them.

---

## Architecture: StreamID and the Stream Table

Every SMMU-managed device transaction carries a **StreamID** — a hardware-fixed identifier assigned to the device (or to a specific channel within the device). The SMMU maintains a **Stream Table** (SMMUv3) or **Context Bank table** (SMMUv2) that maps each StreamID to a translation context.

```
Device issues transaction with StreamID 0x42
        │
        ▼
SMMU looks up Stream Table entry for 0x42
        │
        ├─ STE contains: Stage-2 base address (S2TTB)
        │               Stage-1 context descriptor pointer
        │               Security state (Secure / NonSecure)
        │               ATS configuration
        ▼
Translation executes (S1 if enabled, S2 always for NS streams)
        │
        ▼
Physical address delivered to bus fabric
```

A device with multiple independent DMA engines may have multiple logical channels, each with its own StreamID. This allows different channels of the same physical device to operate in different security domains — for example, a GPU's compute channel and display channel can be isolated from each other.

---

## Two-Stage Translation: The Nested Walk

### Why One Stage Is Not Enough

With single-stage translation, the OS programs the SMMU page tables. But in a virtualised or mixed-criticality system, the OS itself is the entity you do not fully trust. Giving it control of SMMU page tables means giving it the ability to map any device to any physical memory — defeating isolation.

Two-stage translation separates **intent** (what the device wants to access, expressed by the OS) from **permission** (what the device is actually allowed to reach, enforced by the hypervisor).

### The Nested Walk — S2 Is Not a Final Check

The critical property that makes Stage 2 unbypassable: **S2 is invoked at every step of the S1 walk**, not just at the end.

Every pointer in the S1 page table hierarchy is itself an IPA. Before the SMMU can read the next S1 table level, it must resolve that pointer's IPA through Stage 2 to find the physical RAM where that table lives:

```
S1 L1 table pointer (IPA) → S2 check #1 → physical RAM address of L1 table
  └─ SMMU reads L1 entry, finds L2 pointer (IPA)
S1 L2 table pointer (IPA) → S2 check #2 → physical RAM address of L2 table
  └─ SMMU reads L2 entry, finds leaf PTE (IPA output)
S1 leaf output IPA         → S2 check #3 → final physical address

Total: S2 fires 3 times for a 3-level S1 walk.
```

A compromised Guest OS that points its S1 tables at safety-critical memory cannot escape this: even the table pointers are IPAs that must resolve through S2. There is no path through the SMMU walk logic that avoids Stage 2.

### Hypervisor Deployment

Each VM's StreamIDs are mapped to that VM's Stage 2 context. The Stage 2 context contains only that VM's physical memory range. Cross-VM DMA is structurally impossible: a device's StreamID's Stage 2 context has no mapping for another VM's physical memory, regardless of what the Guest OS programs into Stage 1.

**Configuration authority:**
- EL3: programs secure stream mappings — not modifiable by EL1 or EL2
- EL2 (Hypervisor): programs non-secure stream → context mappings and Stage 2 page tables
- EL1 (OS): configures Stage 1 only, within what EL2 delegates

### Hypervisorless AMP Deployment

When no EL2 hypervisor is present (e.g., Linux on Cortex-A + safety RTOS on Cortex-R):

- Stage 1 is **disabled** for DMA masters — not "identity mapped," actually disabled
- EL3 firmware programs Stage 2 directly: each StreamID constrained to its domain's physical range
- EL3 locks SMMU configuration registers (`SMMU_s_CONF`) before launching Linux
- Linux at EL1 cannot reconfigure stream table entries or Stage 2 page tables

This is the same boot-time provisioning and lock pattern as SSC on Cortex-M: security policy configured by the most trusted entity, then the configuration interface is permanently closed for the remainder of the boot cycle.

---

## ATS — Address Translation Services: Performance With a Security Tradeoff

ATS allows a PCIe device to request a translation from the SMMU and cache the result locally in its **ATC (Address Translation Cache)**. Subsequent DMA using the cached translation avoids the SMMU walk latency.

### Full ATS: The Security Gap

In full ATS, the SMMU returns a Host Physical Address (HPA) directly to the device. The device caches it in its ATC. When the device subsequently issues DMA using that cached HPA, the memory controller sees a pre-translated physical address and services the request directly — **without the SMMU re-validating the address**.

This creates a security gap: if the device can modify its own ATC entry (HPA_legit → HPA_forged), it can DMA to arbitrary physical memory that the SMMU's page tables would never have permitted. The SMMU is effectively bypassed for all DMA that uses cached ATS translations.

### Split-Stage ATS: The Mitigation

In split-stage ATS, only the Stage 1 result (IOVA → IPA) is returned to the device. Stage 2 (IPA → HPA) is never exposed to the device. Every DMA using a split-stage ATS translation still goes through Stage 2 inside the SMMU. There is no cached HPA for the device to forge.

The cost: full ATS is required for Peer-to-Peer (P2P) DMA between devices, because P2P DMA requires the initiating device to present an HPA on the PCIe fabric — a GPA cannot be resolved by the receiving device. Full ATS is a necessary feature with a known security cost that must be mitigated through other means (device attestation, SMMU ATS permission tables).

---

## The Lifecycle Dimension — A Fourth Axis

The standard framework treats policy as: Identity × Resource → Allow/Deny.

DMA with ATS reveals a fourth axis: **Lifecycle State**. The security property is not just "who can access what" but "who can access what, for exactly how long, and what must happen in what order before that window closes."

Every DMA attack that operates through stale translations — DMA-after-unmap, ATC cache poisoning, unpin/reuse races — is not a policy violation. The policy was correctly configured. The attack exploits the time gap between policy revocation and hardware enforcement.

Safe revocation requires completing three independent cache invalidations in strict order:

```
① ATC shootdown (PCIe)
   SMMU sends ATS Invalidation Request to device
   Device drains outstanding DMA, clears ATC entry
   Device sends ATS Invalidation Completion
   Host waits for Completion receipt
        ↓
② IOTLB shootdown (MMIO)
   OS writes CMD_TLBI to SMMU command queue
   SMMU invalidates its own IOTLB
   OS writes CMD_SYNC, waits for completion signal
        ↓
③ CPU TLB shootdown (IPI)
   OS calls smp_call_function_many() with cpu_vm_mask
   Each remote CPU receives IPI, executes INVLPG / TLBI
   OS waits for ACK from all target CPUs
        ↓
④ Physical page may now be safely unmapped / reused
```

Skipping or reordering any step creates the window for a stale-translation attack. The ATC step (①) is the most frequently skipped — it requires a PCIe round-trip to the device, which is slow and, in the case of a malicious device, the device may refuse to send a valid Completion.

---

## SMMU vs TZASC — Complementary, Not Alternatives

A common misconception is that SMMU and TZASC protect the same thing in different ways. They protect different points in the same transaction path:

```
Device → [SMMU] → bus fabric → [TZASC] → DRAM controller → DRAM
            ↑                       ↑
  Initiator-side              Target-side
  (between device             (at DRAM controller)
   and bus)
```

| Property | SMMU | TZASC / TZC-400 |
|----------|------|-----------------|
| Position | Between device and bus | Between bus and DRAM controller |
| Side | Initiator-adjacent | Target-side |
| Identity | StreamID (per device/channel) | HNONSEC signal (Secure/NonSecure only) |
| Granularity | Full address space, per-stream policy | Address ranges, binary S/NS |
| Translation | Yes — VA→IPA→PA | No — range check only, no translation |
| DMA coverage | Any device with a StreamID | All initiators regardless of type |
| CPU coverage | No — CPU bypasses SMMU | Yes — TZASC checks all AXI transactions |

The SMMU is powerful but cannot protect against a transaction that doesn't go through it. The TZASC catches everything that reaches DRAM, regardless of initiator — including any path that bypasses the SMMU. Together they provide initiator-side and target-side coverage in depth.

---

## Comparison to SAU and VMIDMT

| Mechanism | Initiator class | Identity | Authority |
|-----------|----------------|----------|-----------|
| SAU (Cortex-M) | CPU transactions | CPU TrustZone state | Secure FW |
| VMIDMT (Qualcomm) | Non-CPU masters | Injected VMID + NS signal | ROT domain |
| **SMMU** | Non-CPU masters | StreamID → translation context | EL1 (S1) / EL2 or EL3 (S2) |
| MPU (ARM) | CPU transactions | CPU privilege level | OS / Secure FW |

SAU and SMMU are the ARM ecosystem's answer to the same question for two different initiator classes. Qualcomm's VMIDMT covers a similar role to SMMU's identity injection for non-CPU masters, but works with XPU (target-side) instead of integrated translation.

---

## Contribution Opportunities

- [ ] SMMUv2 Context Bank format vs SMMUv3 Stream Table — concrete register field breakdown
- [ ] SMMU fault handling — EVT_QUEUE processing in Linux `drivers/iommu/arm/arm-smmu-v3/`
- [ ] PCIe ATS Invalidation protocol — timing and ordering requirements from PCIe spec
- [ ] SVA (Shared Virtual Addressing) — SMMU Stage 1 page tables shared with CPU MMU, PASID binding
- [ ] ARM CCA / RME — Granule Protection Check (GPC) inside SMMU for Realm isolation
- [ ] Qualcomm SMMU deployment — SSD (Security State Determination) signal and multi-ROT context bank ownership
- [ ] Platform-specific: how Linux `arm_smmu_device_probe()` discovers and initialises SMMU instances

---

## References

1. ARM IHI0070 — "ARM System Memory Management Unit Architecture Specification" (normative)
2. Linux kernel `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c` — SMMUv3 driver
3. Linux kernel `Documentation/core-api/cachetlb.rst` — TLB flush interfaces
4. PCIe Base Specification — Address Translation Services (ATS) chapter
5. ACAI: Protecting Accelerator Execution with Arm CCA — SMMU GPC integration (arXiv:2305.15986)
6. Qualcomm Technologies — "An Introduction to Access Control on Qualcomm Snapdragon Platforms" (2020)
7. linuxvox.com — "Who Performs TLB Shootdown?" (Linux IPI shootdown mechanism)
