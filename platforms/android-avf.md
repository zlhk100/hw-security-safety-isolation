# Android AVF (Android Virtualization Framework) — Architecture Analysis

> AVF is designed explicitly for the case where the VMM itself cannot be trusted. crosVM — the Virtual Machine Monitor — runs as an unprivileged Android process (EL0). pKVM at EL2 provides memory isolation not just between VMs, but between each pVM and the Android host kernel at EL1. This is confidential computing without requiring ARM CCA Realm world hardware.

**Status:** Initial content. AVF first shipped Pixel 6 (Android 13). Android 14 broader device support.
**Primary sources:** AOSP documentation (source.android.com), LPC2022 AVF presentation, Android Developers Blog

---

## Framework Mapping

| Dimension | Value |
|---|---|
| **Identity** | Measurement-derived (DICE: CDI = KDF(hardware_UDS, measurement_of_each_layer)) |
| **Resource** | pVM physical memory pages |
| **Enforcement side** | Target — pKVM Stage-2 page tables unmapped from host |
| **Policy authority** | pKVM (EL2) — protects against host kernel (EL1) and crosVM (EL0) |
| **Failure mode** | Stage-2 fault — host kernel cannot access pVM pages |

---

## The Surprising Starting Point: crosVM Is Userspace

The most important fact to establish first: **crosVM, the VMM that creates and manages pVMs, runs as an ordinary Android process at EL0.** It is not trusted. It cannot access pVM memory. Its compromise does not grant access to pVM secrets.

This is the inversion of the traditional VMM trust model. In standard KVM, the VMM (QEMU or crosVM) has broad access to guest memory through the KVM API. In AVF, pKVM at EL2 actively prevents crosVM from reading pVM memory — even though crosVM is the entity that created the VM.

```
EL2  pKVM              — trusted hypervisor
                         manages Stage-2 for both host and pVMs
                         pVM exits arrive here first
                         host kernel CANNOT access pVM Stage-2

EL1  Android/Linux     — untrusted resource manager
                         provides: CPU scheduling, memory allocation, I/O
                         CANNOT read pVM memory (pKVM unmaps Stage-2)

EL0  crosVM            — USERSPACE, untrusted VMM
     VirtualizationService  — manages crosVM lifecycle
     Host App           — communicates with pVM via Binder
```

This is structurally analogous to ARM CCA:
- ARM CCA: RMM (Realm EL2) protects Realm VMs from KVM (Normal EL2)
- pKVM: pKVM (EL2) protects pVMs from both host kernel (EL1) AND crosVM (EL0)

The difference: ARM CCA uses a separate hardware security state (Realm world). pKVM achieves the same isolation within Normal world, on standard ARMv8.2+ hardware without RME.

---

## pKVM: How EL2 Protects Against EL1

In standard KVM, the KVM module is part of the Linux kernel — EL1 and EL2 are the same trusted entity. In pKVM, EL2 has been hardened to protect pVMs against the EL1 host kernel:

pKVM manages ARM Stage-2 page tables for each pVM and for the host kernel. For pVMs, pKVM unmaps the pVM memory region from the host's Stage-2 page tables. The host kernel at EL1 cannot modify these Stage-2 mappings — only pKVM at EL2 can.

```
VM exits from pVM:
  pVM → trap to EL2 (pKVM)
  pKVM evaluates exit reason
    ├─ can handle internally → return to pVM
    └─ needs host kernel (e.g., device I/O) → context-switch CPU state to host EL1

pKVM is the first handler of every pVM exit.
Host kernel only receives exits that pKVM explicitly delegates.
```

This means even VM exits — which in standard KVM go directly to the host kernel — first pass through pKVM, which maintains the isolation invariant at every transition.

---

## pvmfw: The Boot ROM of a pVM

The pVM firmware (pvmfw) is the first code executed by a pVM — analogous to a hardware boot ROM. Its role is to bootstrap the chain of trust and derive the pVM's unique secret, independent of crosVM.

**Boot sequence:**

```
1. Android Bootloader (ABL) loads pvmfw from flash partition
2. ABL verifies pvmfw image integrity (AVB signature)
3. ABL derives DICE CDIs for pvmfw from:
     hardware Root of Trust + measurement(pvmfw)
4. ABL appends CDIs to pvmfw in memory

pKVM protects pvmfw:
  pKVM unmaps pvmfw memory from host Stage-2
  Host kernel and crosVM cannot read pvmfw or its CDIs

5. pVM starts — pKVM hands vCPU control to pvmfw (not guest OS)
6. pvmfw verifies payload:
     checks AVB signature of the OS / Microdroid image
     rejects unsigned or tampered payloads
7. pvmfw derives per-pVM secret via DICE:
     CDI = KDF(CDI_from_ABL, measurement_of_payload)
8. pvmfw wipes secrets from memory
9. pvmfw jumps to verified guest OS
```

crosVM loads the initial memory layout and payload — but pvmfw verifies everything before any guest code runs. crosVM had no opportunity to tamper with the chain of trust.

---

## DICE: The Secret That Is a Function of What Code Is Running

This is the deepest insight in AVF and the one that makes pVMs categorically different from a TEE with a hardware-bound key.

Each pVM has its own secret. The per-pVM secret is not a random number, nor kept in a secure key store. It is a function of measurements of the software that defines the behaviour of the pVM.

```
Legitimate pVM:
  UDS (hardware, from device RoT)
     + measurement(pvmfw)
     + measurement(payload / Microdroid)
     = per-pVM secret S

Attacker replaces payload:
  UDS (same device)
     + measurement(pvmfw)
     + measurement(attacker_payload)   ← different hash
     = per-pVM secret S'              ← DIFFERENT

S' cannot decrypt the instance volume encrypted with S.
Physical replacement of the OS image defeats itself.
```

The same measurement-derived identity principle appears in:
- TDISP/SPDM: device identity includes firmware measurement, not just hardware serial
- ARM CCA: RIM (Realm Initial Measurement) encodes what code is running
- Android AVF: DICE CDI encodes what code is running

All three express the same idea: trust is bound to what is executing, not just to which hardware is present.

---

## The Split-App Programming Model

AVF's developer model is built around one APK containing two code paths:

```
Host APK
├── host/                     ← runs on Android (EL0, untrusted)
│   UI, orchestration, networking
│   VirtualizationService API calls
│   RpcBinder client
│
└── pvm/                      ← runs inside Microdroid pVM (isolated)
    Native .so library
    Implements Binder service
    Sensitive operations:
      - private model inference
      - key derivation
      - biometric matching
      - payment processing
      - DRM content decryption

Communication: RpcBinder over vsock
  Host sends inputs → pVM processes → pVM returns results
  pVM memory: Android host CANNOT access it
```

RpcBinder uses the existing AIDL wire protocol — the developer writes standard Android interfaces. The only change is that the Binder endpoint has moved into a protected VM. Familiar tooling, new security boundary.

---

## AuthFS: Mutual Distrust at the File Boundary

AuthFS is a FUSE filesystem providing integrity-verified file exchange between the host (file server) and pVM (file consumer) at mutually distrusting endpoints.

The host provides files — APKs, APEXes, model weights — to the pVM. But the pVM does not trust the host to provide unmodified content. AuthFS applies Merkle tree verification on every read operation: the pVM can detect if the host tampered with any part of a file between reads.

This is the same principle as the ATC shootdown problem in DMA security: the host controls the storage layer but cannot be trusted to provide unmodified data. Independent verification at the consumer (pVM) catches tampering regardless of host behaviour.

---

## pVM vs TrustZone TEE: Why AVF Exists

Android already has TrustZone with OP-TEE. The question is why AVF adds a new confidential computing layer rather than using the existing TEE.

| Property | TrustZone TEE (OP-TEE TA) | AVF pVM (Microdroid) |
|---|---|---|
| Environment | Constrained C/assembly, GP TEE API | Full Linux, Android NDK, Binder, AIDL |
| Max workload size | Kilobytes to small megabytes | Gigabytes (full ML models, runtimes) |
| Developer tooling | TEE-specific SDK, non-standard | Standard Android development |
| Per-app isolation | Shared Secure world across all TAs | Dedicated pVM per workload |
| Cross-device consistency | Vendor-specific TEE APIs | AVF AIDL (consistent across devices) |
| AI inference | Impractical (memory, toolchain) | Core use case |
| Secret binding | Hardware key (UID, device key) | DICE (measurement-derived) |
| Memory isolation | Hardware (TrustZone Secure world) | pKVM Stage-2 (software EL2) |

The trade-off: TrustZone's Secure world is a separate hardware security state — silicon geometry separates it from Normal world. A pVM's isolation depends on pKVM at EL2 being correct and uncompromised. pKVM has a larger TCB than the TrustZone boundary, but enables workloads that TrustZone's constrained environment cannot support.

---

## What pVM Protection Does Not Cover

**Availability:** pKVM does not protect pVM availability from the host. The host can always preempt or terminate a pVM and reclaim its resources. A compromised host can conduct denial-of-service by terminating pVMs.

**pVM-to-pVM communication:** Direct vsock between pVMs is not permitted. All inter-pVM communication must route through VirtualizationService on the host.

**Only platform-signed apps:** Only apps signed with the platform key can create, own, or interact with pVMs. Not available to arbitrary third-party apps.

**Physical attacks:** pKVM provides no protection against physical DRAM probing — no inline memory encryption equivalent.

**Host-controlled file integrity without AuthFS:** If AuthFS is not used, the host can serve tampered files to the pVM and the pVM cannot detect it.

---

## Hardware Support

| Device | Android version | Notes |
|---|---|---|
| Pixel 6 / 6 Pro | Android 13 | First AVF-capable devices; disabled by default |
| Pixel 7 series | Android 14 | Enabled by default |
| Cuttlefish (emulator) | Android 13+ | Non-protected VMs only |
| ARM RME hardware | Future | pvmfw README: pVM can run in Realm world when RME available |

The pvmfw documentation explicitly states pVMs can run in either "non-secure or realm world" — meaning pKVM-based pVMs are designed to work both on current hardware (Normal world isolation via pKVM EL2) and on future ARM CCA hardware (Realm world isolation via RMM).

---

## Contribution Opportunities

- [ ] pKVM memory management — how pKVM manages Stage-2 donation from host to pVM
- [ ] Secretkeeper HAL — how per-pVM secrets are persisted across device reboots
- [ ] Microdroid manager — complete boot sequence from pvmfw to AIDL service
- [ ] AVF + ARM CCA — what changes when pVM runs in Realm world instead of Normal world
- [ ] Performance overhead — Stage-2 fault handling cost for pVM I/O via shared NS pages
- [ ] MicrodroidDemo — walk-through of the reference split-app implementation on AOSP

---

## References

1. AOSP — "Android Virtualization Framework (AVF) overview" (source.android.com)
2. AOSP — "AVF architecture" (source.android.com/docs/core/virtualization/architecture)
3. AOSP — "Microdroid" (source.android.com/docs/core/virtualization/microdroid)
4. AOSP — "Protected Virtual Machine Firmware" (pvmfw README, android16-release)
5. Google — "Virtual Machine as a core Android Primitive" (Android Developers Blog, Dec 2023)
6. LPC2022 — "Android Virtualization Framework" (lpc.events presentation)
7. ACM SAC 2024 — "Measuring and Optimizing the Performance of the Android Virtualization Framework"
8. Dave Kleidermacher — "Why AVF?" (davek.substack.com, April 2024)
