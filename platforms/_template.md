# [SoC Family Name] — Security Architecture Analysis

> One-line summary of the platform's approach to hardware isolation.

**Status:** [Draft / Review wanted / Complete]
**Reference silicon:** [specific chip names]
**Primary sources:** [datasheet / TRM / vendor security guide]

---

## SoC Overview

<!-- Brief description: CPU cores, bus architecture, key IP blocks.
     Focus on what's relevant to security/safety isolation:
     how many masters, what type of interconnect, DRAM type. -->

---

## Mechanism Inventory

<!-- Complete this table for the platform.
     Be honest about gaps — "unknown" or "not publicly documented" is valid. -->

| Mechanism | Framework Role | Coverage | Authority |
|-----------|---------------|---------|-----------|
| [name] | [Initiator identity / Target enforcer / Both] | [what it protects] | [who programs it] |

---

## DMA Master Coverage

<!-- List every non-CPU bus master on this SoC.
     For each: is there SMMU/IOMMU coverage? MSW/VMIDMT equivalent?
     This is the most common gap in security architectures. -->

| Master | SMMU/IOMMU | Identity mechanism | Notes |
|--------|-----------|-------------------|-------|
| [GPU] | [Yes/No/Partial] | [StreamID / VMID / Master ID] | |

---

## Trust Hierarchy

<!-- Who is the root of trust? How many trust levels?
     Is there a hypervisor? What does EL3 firmware (TF-A or equivalent) do? -->

---

## Lock Sequence

<!-- Step-by-step boot sequence showing when security is configured and locked.
     This is critical for the "hardware-enforced vs software-configured" distinction. -->

```
1. [Boot stage] — [what is configured and locked]
2. ...
```

---

## Gaps and Limitations

<!-- Honest analysis of what this platform's architecture cannot claim.
     Missing coverage, undocumented components, known limitations. -->

---

## Contribution Opportunities

- [ ] ...

---

## References

1. ...
