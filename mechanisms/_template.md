# [Mechanism Name] — [Vendor / Standard]

> One-line description mapping this mechanism to the unified framework.

**Status:** [Draft / Review wanted / Complete]
**Contributed by:** [your GitHub handle]

---

## Framework Mapping

<!-- REQUIRED: Every mechanism page must complete this table -->

| Dimension | Value |
|-----------|-------|
| **Identity signal** | [what identifies the initiating master] |
| **Resource type** | [memory range / peripheral / cache / bandwidth / other] |
| **Enforcement side** | [Initiator / Target / Both] |
| **Policy authority** | [who programs and locks the policy: EL3 / hypervisor / OS / ROT domain] |
| **Failure mode** | [fault / RAZ-WI / reset / bandwidth throttle / other] |

---

## Architecture Overview

<!-- Explain how the mechanism works at the structural level.
     Focus on: where it sits in the bus fabric, what inputs it takes,
     what decision it makes, what happens on violation. -->

---

## Configuration

<!-- Who programs it, when (boot-time / runtime), what the key
     registers or data structures look like.
     Include a concrete example if possible. -->

```c
/* Example configuration code or register pseudocode */
```

---

## Lock Mechanism

<!-- How is the configuration made permanent?
     What register / bit / sequence prevents post-boot modification?
     What is the failure mode if this step is skipped? -->

---

## Comparison to Similar Mechanisms

<!-- How does this mechanism relate to ARM / NXP / Qualcomm / Xilinx equivalents?
     What can it do that they cannot? What are its limitations vs theirs? -->

| Feature | This mechanism | Closest ARM equivalent | Notes |
|---------|---------------|----------------------|-------|
| ... | ... | ... | ... |

---

## Safety / Security Certification Relevance

<!-- Optional but encouraged: what PSA, IEC 61508, ISO 26262,
     or other standard requirement does this mechanism satisfy?
     What evidence does it provide? -->

---

## Contribution Opportunities

<!-- List specific gaps in this page that future contributors could fill.
     Be specific — "add register field breakdown" is better than "add more detail" -->

- [ ] ...

---

## References

<!-- Numbered list. Only public sources:
     datasheets, TRMs, vendor white papers, standards, kernel source, CVEs -->

1. ...
