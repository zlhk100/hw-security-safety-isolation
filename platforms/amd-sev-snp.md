# AMD SEV-SNP — Security Architecture Analysis

> AMD SEV-SNP provides data-in-use protection through two distinct and complementary hardware layers: per-VM memory encryption (confidentiality) and the Reverse Map Table (integrity and anti-aliasing). Understanding that these address different attack classes — and that neither is sufficient alone — is the key insight.

**Status:** Initial content. Contributions welcome.
**Reference silicon:** AMD EPYC 3rd Gen (Milan) and later — first generation with full SNP support
**Primary sources:** AMD SEV-SNP whitepaper (2020), AMD architecture programmer's manual, Linux kernel `arch/x86/kvm/svm/`

---

## Framework Mapping

| Dimension | Layer 1: Encryption | Layer 2: RMP |
|-----------|--------------------|----|
| **Identity signal** | ASID (Address Space ID, per-VM) | ASID + expected GPA per physical page |
| **Resource** | All physical DRAM pages assigned to CVM | Physical 4KB page |
| **Enforcement side** | Target — inline at memory controller | Target — end of Stage-2 MMU walk |
| **Policy** | Encrypt with per-VM key on write; decrypt on read | Check ownership + GPA mapping on every access |
| **Authority** | AMD Secure Processor (PSP) — key management | AMD Secure Processor (PSP) — sole RMP updater |
| **Failure mode** | Ciphertext returned to unauthorised reader | #NPFERROR — handled by PSP, not hypervisor |

---

## The Two-Layer Model — Why Both Are Necessary

This is the most important architectural point: encryption and RMP are not redundant. They defend against different attack classes.

**Layer 1 — Per-VMID Inline Memory Encryption: Confidentiality**

The AMD Secure Processor manages a unique AES-128 key per CVM, identified by ASID. Every memory write by a CVM is encrypted inline by the memory controller before reaching DRAM. Every read is decrypted inline before reaching the CPU.

The hypervisor cannot read CVM memory content because it does not hold the encryption key. Even if the hypervisor physically reads the DRAM pages assigned to a CVM, it sees only ciphertext.

But encryption alone does not provide integrity. A hypervisor that cannot read ciphertext can still:

```
Tamper:  flip bits in encrypted pages
         → guest receives corrupt data it cannot detect

Replay:  swap current encrypted page with older snapshot
         → guest receives stale data indistinguishable from current

Alias:   modify Stage-2 to point a CVM's GPA to a different HPA
         → guest accesses a page belonging to a different VM
         → even though the page is encrypted, it's the wrong page

Remap:   point two different GPAs to the same HPA
         → guest perceives two distinct memory regions
           that are physically the same page
```

None of these attacks are visible to the CVM through encryption alone. The ciphertext the guest receives may be correctly decrypted — it just belongs to a different page, or a different point in time.

**Layer 2 — RMP (Reverse Map Table): Integrity and Anti-Aliasing**

The RMP is a hardware-protected global table maintained by the AMD Secure Processor. It has one entry per 4KB physical page of DRAM. Each entry records:

```
RMP entry for physical page at HPA X:
  ASID         — which CVM owns this page (0 = hypervisor-owned)
  GPA          — the specific guest physical address expected to map here
  Validated    — has the guest explicitly accepted this page assignment
  VMPL[0..3]   — per-VMPL access permissions within the CVM
  PageSize     — 4KB or 2MB
```

When the CPU performs a Stage-2 walk (GPA → HPA) for any CVM memory access, the hardware consults the RMP entry for the resolved HPA. Three checks fire atomically:

```
1. ASID match:    RMP.ASID == currently running CVM's ASID?
2. GPA match:     RMP.GPA == the GPA used in this Stage-2 walk?
3. Validated:     has the guest called PVALIDATE on this page?

Any check fails → #NPFERROR
  Handled by AMD SP firmware, not by the hypervisor
  Hypervisor cannot intercept or suppress this fault
```

This closes all four attack vectors:

| Attack | How RMP closes it |
|---|---|
| Tamper (bit flip) | AES-GCM authentication tag detects modification |
| Replay (old snapshot) | New Stage-2 mapping → different HPA → RMP.GPA mismatch |
| Alias (wrong physical page) | RMP.ASID or RMP.GPA mismatch → fault |
| Remap (two GPAs → one HPA) | RMP.GPA records the one authorised GPA; second mapping → fault |
| Cross-VM page theft | RMP.ASID still holds original VM's ASID → fault |

---

## The PSP: Sole RMP Management Authority

The hypervisor **cannot directly write RMP entries**. This is the structural guarantee that makes SNP robust even against a compromised hypervisor.

All page operations flow through PSP-mediated interfaces:

```
Hypervisor requests page assignment → calls SNP firmware command
  ↓
AMD SP (PSP) validates the request
AMD SP updates RMP entry
AMD SP returns result to hypervisor

Hypervisor cannot:
  - write RMP entries directly
  - suppress RMP faults
  - read CVM register state
  - access pages marked with another VM's ASID
```

This is structurally identical to the pattern seen across every confidential computing architecture:

| Architecture | Protection table | Sole authority |
|---|---|---|
| AMD SEV-SNP | RMP | AMD Secure Processor (PSP) |
| ARM CCA | GPT (Granule Protection Table) | Root Monitor (EL3) |
| Android pKVM | Stage-2 page tables | pKVM (EL2) |
| TDISP | IOMMU S2 context | TSM (TEE Security Manager) |

The entity managing resources (hypervisor, KVM) is always separated from the entity managing protection (PSP, Root Monitor, pKVM, TSM).

---

## VMPL: Intra-VM Privilege Levels

SEV-SNP introduced VMPL (Virtual Machine Privilege Levels) — four privilege levels (VMPL0 through VMPL3) within a single CVM:

```
VMPL0  most privileged — can act as security monitor inside the CVM
VMPL1  intermediate
VMPL2  intermediate
VMPL3  least privileged — unprivileged userspace equivalent
```

The RMP entry carries per-VMPL permission bits for each physical page. VMPL0 can restrict which pages VMPL1–3 can access. This enables a trust hierarchy inside the CVM itself:

- VMPL0: in-VM attestation agent or security monitor (e.g., Coconut-SVSM)
- VMPL1: guest OS kernel
- VMPL3: guest applications

This is structurally analogous to ARM CCA's Realm world having R-EL2 (RMM) and R-EL1 (guest kernel) as separate privilege levels within the Realm — a trusted component at the highest in-VM privilege level managing protection for less privileged components below it.

---

## RMP vs GPT/GPC: The Same Pattern at Different Granularity

Both AMD SEV-SNP RMP and ARM CCA GPT/GPC implement the same fundamental pattern: a hardware-protected per-page ownership table, checked at the output of address translation, that the untrusted hypervisor cannot modify.

| Property | AMD SEV-SNP RMP | ARM CCA GPT/GPC |
|---|---|---|
| Granularity | 4KB per physical page | 4KB per physical granule |
| Identity | ASID (per-VM) | Security world (Realm/NS/Secure/Root) |
| Authority | AMD SP (PSP) | Root Monitor (EL3) |
| Check point | End of Stage-2 MMU walk | Output of every Stage 1+2 MMU/SMMU translation |
| DMA coverage | IOMMU checks RMP on device DMA | SMMU GPC checks GPT on device DMA |
| In-VM hierarchy | VMPL 0–3 | R-EL2 (RMM) / R-EL1 / R-EL0 |
| Memory encryption | Mandatory (AES-128 per-VM key) | Implementation-defined (not mandated) |

Both are target-side mechanisms. Both operate at 4KB granularity. Both make the same architectural claim: the untrusted hypervisor that allocates the pages cannot access or remap them after the CVM owns them. The difference is execution: AMD does it through a firmware-managed table with explicit ownership records; ARM does it through a hardware-enforced world-membership attribute.

---

## Attestation — TCG RATS Model

AMD SEV-SNP attestation follows the TCG RATS three-role model (RFC 9334):

```
Endorser (AMD)                RVP (workload / TCB vendor)
provides: VCEK cert chain     provides: expected measurements
         ↓                             ↓
         └──────────────┬──────────────┘
                         ▼
              Verifier (AMD KDS + tenant-side service)
                consults VCEK certificate chain
                compares measurements against RVP
                produces Attestation Result
                         ↓
              Relying Party (tenant)
                receives Attestation Result only
                never verifies raw Evidence directly
                provisions secrets to CVM if AR passes
```

**VCEK (Versioned Chip Endorsement Key):** Per-chip certificate derived from the AMD root key, encoding the current TCB version (microcode, SNP firmware, boot firmware). The VCEK changes when any TCB component is updated. The Relying Party uses the VCEK version in the Attestation Result to verify the platform is running a trusted TCB version.

**ReportedTCB:** The set of TCB version numbers included in the attestation report, signed by the VCEK. The Verifier compares these against the Relying Party's minimum acceptable TCB policy.

---

## Contribution Opportunities

- [ ] VMPL0 use cases — Coconut-SVSM as an in-VM security monitor; paravisor pattern
- [ ] AMD IOMMU + RMP interaction — how DMA to SNP-protected pages is enforced
- [ ] SNP firmware commands — LAUNCH_START, LAUNCH_UPDATE, LAUNCH_MEASURE, LAUNCH_FINISH sequence
- [ ] VCEK certificate hierarchy — AMD root CA → ARK → ASK → VCEK
- [ ] Migration of SNP VMs — how page ownership transfers during live migration
- [ ] Comparison with Intel TDX — SEPT (Secure EPT) as TDX's equivalent to RMP

---

## References

1. AMD — "AMD SEV-SNP: Strengthening VM Isolation with Integrity Protection and More" (2020)
2. AMD Architecture Programmer's Manual — SEV-SNP chapter
3. Linux kernel `arch/x86/kvm/svm/sev.c` — SNP hypervisor implementation
4. IETF RFC 9334 — Remote Attestation Procedures (RATS) Architecture
5. AMD KDS — Key Distribution Service for VCEK certificates
6. Coconut-SVSM — VMPL0 security monitor reference implementation
