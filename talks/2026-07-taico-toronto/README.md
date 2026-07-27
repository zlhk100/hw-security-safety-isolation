# Sovereign AI Is a Hardware Problem

**Event:** TAICO Toronto · July 22, 2026
**Speaker:** Lei Zhou — Systems Architect, Platform Security Practitioner
**Format:** 30-minute independent practitioner talk

---

## Abstract

Most sovereign AI conversations stop at three choices: regional cloud provider, data residency policy, open-weight model. These are the right starting points. They are also incomplete in a specific way that only becomes visible from below the API gateway.

This talk traces what deploying a three-tier sovereign LLM stack across EU, NA, and Asia-Pacific jurisdictions revealed about what "sovereignty" actually means at the hardware layer — and why the gap between policy promise and cryptographic proof matters for any workload where the answer to "can you attest to what's actually running?" needs to be yes.

The structural insight at the centre: two engineering communities — ISO 26262 hardware architects and AI security researchers — have independently arrived at the same conclusion through different routes. Enforcement cannot live in the same failure domain as what it enforces. For functional safety engineers, this is a formal requirement (Freedom from Common-Cause Failures). For AI security researchers, it is an empirical finding from broken guardrails and LLM judges that do not hold. Neither community appears to cite the other.

---

## Slides

[sovereign-ai-hardware-problem.pdf](./sovereign-ai-hardware-problem.pdf)

---

## Key Topics

- Why data residency policy and hardware sovereignty are different questions
- The three gaps that appear in MaaS-based inference when viewed from below the API
- A three-tier trust architecture: provider policy → architectural enforcement → cryptographic proof
- The protection matrix: four deployment architectures across nine sovereignty dimensions
- Confidential Computing, Measured Boot, and Remote Attestation (SPDM / DICE) as the convergence point
- The ISO 26262 CCF / AI security structural parallel

---

## Related Library Entries

- [`platforms/arm-cca.md`](../../platforms/arm-cca.md) — Arm Confidential Compute Architecture
- [`platforms/amd-sev-snp.md`](../../platforms/amd-sev-snp.md) — AMD SEV-SNP
- [`use-cases/confidential-computing-trusted-io.md`](../../use-cases/confidential-computing-trusted-io.md) — Confidential Computing and trusted I/O
- [`use-cases/sdv-mixed-criticality.md`](../../use-cases/sdv-mixed-criticality.md) — Mixed-criticality systems and the CCF independence requirement

---

## Follow-up

LinkedIn post from this talk: [Enforcement cannot live in the same failure domain as what it enforces](https://www.linkedin.com/in/leizhou)

Next talk in this series: *Closing the Gap* — Confidential Computing, SPDM, and Remote Attestation in depth.

---

*Lei Zhou · [github.com/zlhk100](https://github.com/zlhk100) · [linkedin.com/in/leizhou](https://linkedin.com/in/leizhou) · [medium.com/@zlhk100](https://medium.com/@zlhk100)*
