# ARM CCA (Confidential Compute Architecture) — Security Architecture Analysis

> ARM CCA introduces a fourth hardware security state — Realm world — alongside the existing Normal, Secure, and new Root worlds. Realm VMs running in Realm world are the CVMs. Normal world guest VMs are not CVMs. The distinction is absolute: a NS guest VM cannot become a CVM while remaining in Normal world.

**Status:** Initial content. Hardware appearing in Neoverse V3, Cortex-A720.
**Standards:** ARM Den0125 (CCA Software Architecture), ARM Den0125 (CCA Hardware Architecture), IETF RFC 9334

---

## Framework Mapping

| Mechanism | Identity | Resource | Side | Authority |
|---|---|---|---|---|
| GPT/GPC | Security world (Root/Realm/Secure/NS) | Physical granule (4KB) | **Target** — output of every MMU+SMMU translation | Root Monitor (EL3) exclusively |

GPT/GPC is target-side access control at 4KB granularity — the same pattern as MPC and TZASC, with four-world identity instead of binary Secure/NonSecure, and per-page granularity instead of address ranges.

---

## The Four-World Model

ARM CCA extends TrustZone's two-world model to four worlds, each with its own exception levels:

```
Root world    (EL3 only)
  TF-A Monitor
  Sole authority over GPT updates
  Handles world switching and RMI dispatch
  No application code runs here

Normal world  (EL0/EL1/EL2)
  KVM/Linux hypervisor (EL2)    — resource manager
  Regular QEMU guest VMs (EL1)  — untrusted, standard isolation
  Host applications (EL0)

Realm world   (R-EL0/R-EL1/R-EL2)
  RMM — Realm Management Monitor (R-EL2)   — protection enforcer
  Realm VM guest kernel (R-EL1)            — the CVM
  Realm VM applications (R-EL0)

Secure world  (S-EL0/S-EL1/S-EL2)
  TEE OS (OP-TEE, Hafnium) — unchanged from TrustZone
  Trusted Applications
```

**The critical architectural point:** Realm VMs running in Realm world ARE the CVMs. Normal world guest VMs are standard VMs with no confidential computing properties — KVM can fully access their memory. A NS guest VM cannot be promoted to a CVM without moving to Realm world.

---

## GPT/GPC: The Same Target-Side Pattern at 4KB Granularity

The Granule Protection Table (GPT) is controlled exclusively by the Root Monitor at EL3. It assigns each 4KB physical granule a world membership:

```
GPI_NS       → accessible from Normal world
GPI_REALM    → accessible from Realm world only
GPI_SECURE   → accessible from Secure world only
GPI_ROOT     → accessible from Root world only
GPI_NONSECURE → shared: accessible from Normal and Realm (I/O buffers)
```

The Granule Protection Check (GPC) fires at the **output of every address translation** — Stage 1, Stage 2, and SMMU. When any translation produces a physical address, the GPC looks up that PA in the GPT. If the accessing world does not match the GPT entry, a Granule Protection Fault occurs.

This applies to both CPU accesses and DMA: the SMMU's GPC extension checks GPT on every translated IOMMU PA, preventing device DMA from reaching Realm world memory from a Normal world device context.

**The "aha" connection to the framework:** GPT/GPC is not a new security pattern. It is MPC/TZASC at per-page granularity with four-world identity. Confidential computing extended the granularity and enriched the identity model — it did not invent a new protection pattern.

---

## Two Hypervisors, Two Jobs

ARM CCA's architecture specification states the design principle explicitly:

- **Policy** (which resources to allocate to a Realm): responsibility of KVM in Normal EL2
- **Mechanism** (enacting changes that cannot be trusted to the hypervisor): responsibility of RMM in Realm EL2

```
KVM (Normal EL2)          RMM (Realm EL2)
Resource manager          Protection enforcer
Allocates: CPU, memory    Manages: RTTs (Realm Translation Tables)
Creates Realm VMs         Handles RMI requests
Schedules Realm vCPUs     Processes RSI from guest
CANNOT read Realm memory  Measured + included in attestation
```

KVM retains **control plane authority** — it decides what resources a Realm gets, creates and destroys Realms, schedules vCPUs. KVM loses **data plane authority** — once pages are delegated to Realm world, KVM cannot read, write, or remap them.

---

## Memory Delegation: One-Way During VM Lifetime

When KVM assigns physical pages to a Realm VM, it calls `RMI_GRANULE_DELEGATE` via SMC:

```
KVM (Normal EL2)
  │  SMC: RMI_GRANULE_DELEGATE(page_PA)
  ▼
Root Monitor (EL3)
  │  Updates GPT: page_PA → GPI_REALM
  │  Switches to Realm EL2
  ▼
RMM (Realm EL2)
  │  Receives delegation, assigns to Realm VM RTTs
  ▼
  Page is now Realm PAS
  Any Normal world access → GPC fault immediately
  KVM cannot read, write, or remap this page
```

Memory flows Normal → Realm but not back while the Realm is running. KVM can destroy the Realm at any time, which forces undelgation back to Normal PAS. KVM retains lifecycle authority (create/destroy) and loses memory access authority during the Realm's lifetime. This is the boot-time-lock pattern — the same principle as SSC, HDP, and SMMU_s_CONF — applied per-page at runtime rather than once at boot.

---

## The Realm VM Controls Its Own I/O Boundary

Because KVM cannot access Realm memory, standard virtio I/O cannot work directly. The solution inverts the traditional authority model: the Realm VM decides which pages to share with the hypervisor, not the other way around.

Using RSI (Realm Services Interface) — calls from Realm R-EL1 to RMM at R-EL2 — the guest explicitly transitions specific pages to GPI_NONSECURE for device I/O:

```
Realm VM: RSI_IPA_STATE_SET(buffer_page, NS)
RMM: SMC to Root Monitor
Root Monitor: GPT update → buffer_page becomes GPI_NONSECURE
KVM: can now DMA device ↔ buffer_page
```

KVM cannot pull pages out of the Realm. The Realm pushes pages out when it needs I/O. Authority is inverted relative to traditional virtualisation.

---

## What KVM Can and Cannot Do

| Operation | KVM can? | Enforcement |
|---|---|---|
| Create a Realm VM | ✅ | KVM calls RMI_REALM_CREATE via SMC |
| Schedule Realm vCPUs | ✅ | KVM calls RMI_REC_ENTER |
| Assign physical pages | ✅ (delegate) | RMI_GRANULE_DELEGATE → GPT update by Root Monitor |
| Provide virtio I/O | ✅ (via NS-shared pages) | Realm VM must RSI-share pages first |
| Destroy Realm VM | ✅ (any time) | Forces undelgation of all pages |
| Read Realm VM memory | ❌ | GPT = GPI_REALM → GPC fault |
| Modify Realm page tables | ❌ | RTTs managed exclusively by RMM |
| Observe Realm register state | ❌ | Protected during world switch |
| Forge attestation token | ❌ | Signed by Root Monitor with platform key |

---

## Attestation — TCG RATS Three-Role Model

ARM CCA attestation follows RFC 9334. The Relying Party never verifies raw Evidence directly.

```
Endorser (ARM + silicon mfr)     RVP (RMM vendor + workload vendor)
ARM root endorsement key          expected RMM digest
per-device platform certificate   expected Realm VM image digest
          │                                │
          └──────────────┬─────────────────┘
                          ▼
               Verifier (e.g., ARM Veraison)
                 verifies platform token signature (ARM cert chain)
                 verifies RAK binding to attested platform
                 compares RIM against RVP reference values
                 compares RMM measurement against RVP
                 produces Attestation Result
                          ↓
               Relying Party (tenant)
                 receives Attestation Result only
                 never sees raw CCA token (CBOR/COSE)
                 provisions secrets to Realm VM if AR passes
```

**Attestation data produced by the Realm VM:**

```
Realm VM calls RSI_ATTESTATION_TOKEN_INIT(nonce from Verifier)
RMM collects:
  RIM  — Realm Initial Measurement (hash of initial Realm image)
  REM  — Realm Extensible Measurements (runtime measurements)
  RMM measurement (from Root Monitor's boot-time measurement of RMM)
  Platform measurements
Root Monitor signs CCA token with platform private key

TCB in the attestation token:
  CPU hardware (RME extension)
  Root Monitor (TF-A, measured at boot)
  RMM (measured, included in token)
  Realm VM initial image (RIM)

NOT in TCB:
  KVM — removed from trust path entirely
  Host Linux
  Other Realm VMs
```

---

## Memory Encryption: Implementation-Defined

ARM CCA hardware architecture states isolation is enforced "by faulting exceptions and by physical memory encryption." However, **memory encryption is implementation-defined — not architecturally mandated**.

A compliant CCA implementation can exist without inline DRAM encryption, relying solely on GPT/GPC for isolation. This contrasts with AMD SEV-SNP where per-VM AES-128 encryption is a mandatory architectural feature.

For deployments where physical DRAM probing is in the threat model, the presence or absence of inline memory encryption is a platform selection criterion — not guaranteed by the ARM CCA specification itself.

---

## Comparison: ARM CCA vs AMD SEV-SNP vs Intel SGX

| Property | ARM CCA (Realm VM) | AMD SEV-SNP | Intel SGX |
|---|---|---|---|
| Isolation granularity | VM-level (Realm world) | VM-level | Process-level (enclave) |
| Page ownership table | GPT/GPC (4KB, 4 worlds) | RMP (4KB, per-ASID + GPA) | EPCM (4KB, per-enclave) |
| Table side | Target | Target | Target (processor-internal) |
| Hypervisor in TCB? | No (KVM excluded) | No (hypervisor excluded) | N/A |
| Trusted component | RMM (R-EL2) + Root Monitor (EL3) | AMD SP (PSP) | CPU enclave mode |
| Memory encryption | Implementation-defined | Mandatory (AES-128/256) | Mandatory (MEE) |
| Intra-VM privilege | R-EL2/EL1/EL0 | VMPL 0–3 | N/A |
| Device DMA to CVM | SMMU GPC + TDISP (in development) | AMD IOMMU + RMP checks | Not supported (EPC is CPU-internal) |
| Entry point control | World switch (hardware) | VMGEXIT | ECALL via EENTER (SG equivalent) |

---

## Silicon Availability (late 2025)

ARM CCA / RME is no longer purely prospective:
- Neoverse V3 — first server-class ARM core with RME support
- Cortex-A720 — first application-class core with RME support
- Linux kernel CCA/KVM patch series active (arm64 CCA support merged)

---

## Contribution Opportunities

- [ ] RMM internals — how RMM manages RTTs (Realm Translation Tables) for multiple Realm VMs
- [ ] ARM Veraison — the reference Verifier implementation for CCA attestation
- [ ] SMMU GPC extension — how GPC integrates into SMMU for device DMA coverage of Realm memory
- [ ] CCA + TDISP — how TDISP device attestation integrates with Realm VM attestation
- [ ] Neoverse V3 platform — first production CCA silicon architecture details
- [ ] CCA vs TDX — detailed comparison of attestation token formats and TCB structures

---

## References

1. ARM Den0125 — "Arm CCA Software Architecture" (April 2025)
2. ARM Den0125 — "Arm CCA Hardware Architecture" — GPT, GPC, Realm EL2
3. Linux kernel Documentation/arch/arm64/arm-cca.rst
4. Linux kernel KVM CCA patch series — "[PATCH v3 00/43] arm64: Support for Arm CCA in KVM"
5. IETF RFC 9334 — Remote Attestation Procedures (RATS) Architecture
6. ACAI — "Protecting Accelerator Execution with Arm CCA" (arXiv:2305.15986)
7. virtCCA — "Enabling Confidential Computing in Arm-based Virtualized Environments" (arXiv:2306.11011)
8. ARM Veraison — reference Verifier for CCA attestation: github.com/veraison
