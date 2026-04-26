# [Use Case Title]

> One-line description of the safety/security problem and the hardware solution.

**Status:** [Draft / Review wanted / Complete]
**Domain:** [Automotive / Robotics / Industrial / Medical / IoT]
**Standards:** [ISO 26262 / IEC 61508 / IEC 62443 / PSA / other]

---

## Problem Statement

<!-- What safety or security requirement is being addressed?
     What are the two (or more) components that must be isolated?
     What shared resources create the interference risk?
     Why is software-only isolation insufficient? -->

---

## Hardware Architecture

<!-- Which mechanisms are used and why.
     Show the isolation boundary concretely — which StreamIDs,
     which PARTIDs, which memory regions, which XPU resource groups.
     ASCII diagrams or tables are fine. -->

### Spatial Isolation

### Temporal Isolation (if applicable)

---

## The Safety Argument

<!-- Complete the argument in the form regulators / assessors expect:
     "Component X cannot interfere with Component Y because [hardware mechanism]
     enforces [specific constraint] and this is [permanent / locked / verified]." -->

### Spatial FFI Claim

### Temporal FFI Claim (if applicable)

---

## The Security Argument (if applicable)

<!-- What threat model does this architecture address?
     What attack path does the hardware enforcement block? -->

---

## Boot Sequence

<!-- Step-by-step: who configures what, in what order, and what is locked when. -->

---

## Gaps and Open Questions

<!-- Honest analysis of what this architecture does NOT yet address.
     Open certification questions. Platform-specific unknowns. -->

---

## Contribution Opportunities

- [ ] ...

---

## References

1. ...
