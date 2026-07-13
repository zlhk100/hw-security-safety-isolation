# Intel SGX — Security Architecture Analysis

> Intel SGX provides process-level confidential computing through hardware-enforced enclave isolation. The EPCM (Enclave Page Cache Map) is SGX's target-side page ownership mechanism — structurally identical to AMD's RMP and ARM CCA's GPT, at process rather than VM granularity. SGX is now primarily a data-centre technology; it has been deprecated on most Intel consumer platforms.

**Status:** Initial content. SGX1/SGX2 architecture. Platform availability note important.
**Reference silicon:** Intel Xeon Scalable (Ice Lake-SP and later) for production SGX
**Primary sources:** Intel SGX Developer Guide, Intel SDM Volume 3D

---

## Framework Mapping

| Dimension | Value |
|---|---|
| **Identity** | Enclave instance (SECS — SGX Enclave Control Structure) |
| **Resource** | Enclave pages within EPC (Enclave Page Cache) |
| **Enforcement side** | Target — processor-internal, checked on every EPC access |
| **Policy authority** | CPU enclave mode (hardware-internal) — no external firmware |
| **Failure mode** | #GP fault — access to EPC page from wrong enclave or non-enclave code |

---

## Platform Availability — Important Note

**SGX has been deprecated on Intel consumer platforms since Ice Lake (2021).** It is no longer supported on:
- Intel 12th Gen Core (Alder Lake) and later — consumer desktop/laptop
- Most Intel NUC and embedded platforms

SGX is actively supported on:
- **Intel Xeon Scalable (Ice Lake-SP, Sapphire Rapids, Granite Rapids)** — data-centre CPUs
- Intel TDX has replaced SGX for VM-level confidential computing on newer Xeon platforms

For new designs requiring confidential computing at process granularity on Intel silicon, verify SGX availability on the specific Xeon SKU. For VM-level isolation, Intel TDX is the current direction.

---

## Architecture Overview

### EPC and EPCM: The Core Mechanism

SGX carves out a contiguous physical memory region called the **PRM (Processor Reserved Memory)** from system DRAM. Within the PRM sits the **EPC (Enclave Page Cache)** — the physical pages available for enclave use. The EPC is managed by the processor and inaccessible to the OS or hypervisor.

The **EPCM (Enclave Page Cache Map)** is the hardware-maintained ownership table — one entry per EPC page:

```
EPCM entry for EPC page at PA X:
  VALID      — is this page assigned to any enclave?
  PT         — page type: REG (regular), SECS, TCS, VA, ...
  ENCLAVESECS — pointer to the SECS of the owning enclave
  ENCLAVEADDRESS — the linear address within the enclave this page maps to
  R, W, X    — read/write/execute permissions for this page
  BLOCKED    — page in process of being evicted
```

On every access to an EPC page, the processor checks the EPCM:
- Is the accessor the enclave that owns this page (ENCLAVESECS match)?
- Is the linear address used to access the page the authorised address (ENCLAVEADDRESS match)?
- Does the access type (R/W/X) match the EPCM permissions?

Any mismatch → #GP fault. No software can suppress or intercept this check.

**Framework connection:** EPCM is target-side page ownership enforcement. Same pattern as AMD RMP and ARM CCA GPT/GPC — hardware-maintained per-page ownership table checked at the point of access, that no privileged software can bypass.

---

## Memory Encryption: MEE (Memory Encryption Engine)

EPC pages are encrypted by the MEE (Memory Encryption Engine) — a hardware block between the CPU cache hierarchy and DRAM controller. All EPC data is encrypted with an ephemeral key generated at power-on and never exposed to software. The key is discarded on power-off.

Unlike AMD SEV-SNP's per-VM keys or ARM CCA's optional encryption, SGX's MEE encryption is:
- Mandatory for all EPC pages (no software control)
- Ephemeral (new key each boot — no persistence after power-off by design)
- Integrity-protected (MEE uses a Merkle tree for replay and tamper detection)

The Merkle tree is the SGX equivalent of the RMP's GPA match check: it detects if a physical DRAM page has been tampered with or swapped since it was last written by the enclave.

---

## The ECALL Gateway: The NSC/SG Parallel

SGX defines the only legal way for non-enclave code to call into an enclave: through an **ECALL** (Enclave Call). ECALLs must enter the enclave at a registered entry point defined in the enclave's **TCS (Thread Control Structure)**.

Any attempt to branch into arbitrary enclave code from outside — not through a defined TCS entry point — results in a fault.

This is structurally identical to ARM Cortex-M33's NSC/SG mechanism:

```
SGX ECALL                        ARM Cortex-M33 NSC/SG
────────────────────             ────────────────────────────
TCS defines valid entry points   NSC region contains SG stubs
EENTER targets TCS               Branch must land on SG instruction
Arbitrary branch into EPC → #GP  Arbitrary branch into Secure → SecureFault
Hardware-enforced API boundary   Hardware-enforced API boundary
```

Both are hardware-enforced controlled entry points into a protected domain. Both cause a fault on arbitrary cross-domain branches. The difference: SGX enforces at the instruction level (EENTER must target a valid TCS); ARM NSC enforces at the branch target (must land on SG instruction). Same security property, different implementation mechanism.

---

## Enclave Lifecycle: Create, Populate, Measure, Initialize

```
1. ECREATE
   OS allocates EPC pages for the SECS
   Processor creates enclave: initialises SECS, assigns MRENCLAVE measurement register

2. EADD (repeated for each enclave page)
   OS adds EPC pages to the enclave
   Processor assigns each page: updates EPCM entry
   Page type: REG (code/data), TCS (thread control structure)

3. EEXTEND (for each 256-byte chunk to be measured)
   Processor extends MRENCLAVE: SHA-256 hash of page content
   Builds cumulative measurement of all enclave content

4. EINIT
   Requires SIGSTRUCT (vendor signature) and EINITTOKEN (launch token)
   Processor verifies signature, finalises MRENCLAVE and MRSIGNER
   Enclave is now initialised — EPCM entries locked

5. EENTER (runtime)
   Non-enclave code calls EENTER targeting a valid TCS
   Processor switches to enclave mode, transfers control to entry point

6. EEXIT / AEX (runtime)
   EEXIT: enclave voluntarily returns control to non-enclave code
   AEX (Asynchronous Enclave eXit): hardware interrupt/exception exits enclave
```

SGX2 adds **EAUG** — dynamic EPC page addition after EINIT — enabling enclaves to request additional pages at runtime without restarting.

---

## Remote Attestation

SGX uses an Intel-provisioned attestation infrastructure:

```
Enclave generates REPORT (local attestation structure):
  MRENCLAVE  — measurement of enclave code and data
  MRSIGNER   — measurement of enclave signing key
  ISVPRODID  — product ID
  ISVSVN     — security version number
  REPORTDATA — 64 bytes of enclave-provided data (e.g., public key)

Intel DCAP (Data Center Attestation Primitives) flow:
  Enclave → Quote (signed REPORT) via Quoting Enclave
  Quote → Verifier
  Verifier consults Intel PCK cert chain (Endorser)
  Verifier checks MRENCLAVE against expected value (RVP)
  Verifier produces Attestation Result
  Relying Party receives AR (never raw Quote)
```

This follows the TCG RATS three-role model. The Relying Party never directly verifies the Quote — it delegates to a Verifier service (Intel Trust Authority, or operator-deployed DCAP Verification Service).

---

## SGX vs TDX: Intel's Two Confidential Computing Approaches

| Property | Intel SGX | Intel TDX |
|---|---|---|
| Isolation granularity | Process (enclave) | VM (Trust Domain) |
| Protected unit | EPC pages within enclave | Trust Domain (TD) memory |
| Host OS/VMM access | Cannot access EPC | Cannot access TD memory |
| Page ownership | EPCM (per-EPC-page) | SEPT (Secure EPT, per-TD-page) |
| Memory encryption | Ephemeral key (MEE) | Per-TD key (MKTME) |
| Workload type | Small, security-critical computation | Full OS, rich environment |
| Device DMA to protected memory | Not supported (EPC is CPU-internal) | Via TDISP (Intel TDX Connect) |
| Developer environment | Custom SGX SDK | Standard Linux VM |

SGX and TDX are complementary, not competing: SGX for process-level fine-grained isolation of specific computations; TDX for VM-level isolation of complete workloads including OS and runtime.

---

## Contribution Opportunities

- [ ] Intel TDX deep-dive — SEPT as TDX's equivalent of RMP/GPT; TD lifecycle; TDX Connect + TDISP
- [ ] SGX EPC eviction — what happens when EPC capacity is exceeded; VA page mechanism
- [ ] SGX sealing — how enclaves persist data across restarts using EGETKEY + AES-GCM
- [ ] SGX2 EDMM — dynamic memory management in SGX2; EAUG, EMODT, EMODPR
- [ ] Intel DCAP — deployment without Intel's online attestation service; PCK cert caching
- [ ] SGX side-channel history — Spectre, LVI, SGAxe — what architectural mitigations exist

---

## References

1. Intel — "Intel SGX Developer Guide" (current)
2. Intel SDM Volume 3D — SGX Instructions Reference
3. Intel — "Intel TDX Connect Architecture Specification" (2023)
4. IETF RFC 9334 — Remote Attestation Procedures (RATS) Architecture
5. Intel — "Intel SGX Deprecation on 6th to 10th Generation Intel® Core™ Processors" (2021)
6. Intel DCAP — Data Center Attestation Primitives documentation
