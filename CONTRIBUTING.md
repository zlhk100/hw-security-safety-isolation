# Contributing to hw-security-safety-isolation

Thank you for contributing. This library grows through engineering community knowledge — corrections, new mechanisms, platform analyses, and safety certification case studies are all welcome.

---

## What We're Looking For

### High-value contributions

- **New mechanism pages** (`mechanisms/`) — any hardware protection component not yet covered: RISC-V IOPMP, Apple DART, MediaTek EMI MPU, TI Firewall, Intel VT-d, ARM CCA, etc.
- **Platform analyses** (`platforms/`) — per-SoC security architecture breakdowns mapping vendor mechanisms to the unified framework
- **Safety/security co-design patterns** (`use-cases/`) — concrete FFI arguments, joint certification strategies, mixed-criticality SoC design patterns
- **Corrections** — technical inaccuracies, outdated register names, incorrect mechanism classifications
- **Diagram improvements** — cleaner SVGs, additional interactive diagrams, new visual explanations

### We are not looking for

- Marketing content or vendor promotion
- Content that cannot be sourced to a public reference
- Mechanism descriptions that don't map to the Identity × Resource → Policy framework
- NDA-protected or proprietary information

---

## Contribution Rules

### 1. Accuracy over completeness

Every technical claim must be traceable to a public source:
- Vendor datasheet or technical reference manual
- Vendor white paper (as in the Qualcomm section)
- ARM Architecture Reference Manual
- Published standard (IEC 61508, ISO 26262, PSA scheme documents)
- Peer-reviewed paper or conference proceeding
- CVE database entry (for vulnerability examples)

If you are uncertain about a claim, mark it explicitly: `> **Note:** This is inferred from [source] — verification welcome.`

### 2. Framework-aligned structure

Every new mechanism page should answer the four-part template. Use this as your opening section:

```markdown
## Framework Mapping

| Dimension | Value |
|-----------|-------|
| **Identity signal** | [what identifies the initiator] |
| **Resource type** | [what is being protected] |
| **Enforcement side** | Initiator / Target / Both |
| **Policy authority** | [who programs and locks the policy] |
| **Failure mode** | [fault / RAZ-WI / reset / bandwidth enforcement] |
```

### 3. No NDA-protected content

All content must be based on publicly available sources. Do not include information from NDAs, confidential customer engagements, or pre-release documentation. If in doubt, leave it out.

### 4. Vendor-neutral framing

Mechanisms from any vendor are welcome. The framing should be analytical, not promotional. "Qualcomm's XPU is equivalent to ARM's MPC but with the addition of APU mode for fragmented ranges" is good. "Qualcomm's XPU is the industry-leading solution for access control" is not.

### 5. Safety claims need standard references

Claims about FFI, SIL levels, or certification evidence must reference the relevant standard (IEC 61508, ISO 26262, ISO 13849, IEC 62443, PSA scheme documents). Do not make certification claims without grounding them in normative requirements.

---

## File Structure for New Contributions

### New mechanism (`mechanisms/`)

```markdown
# [Mechanism Name] — [Vendor/Standard]

> One-line description mapping to the framework.

## Framework Mapping
[table as above]

## Architecture
[how it works — initiator side, target side, or both]

## Configuration
[who programs it, when, what the registers look like]

## Lock Mechanism
[how the configuration is made permanent]

## Failure Mode
[what happens on a policy violation]

## Comparison to Similar Mechanisms
[how it relates to ARM/NXP/Qualcomm equivalents]

## References
[numbered list of public sources]
```

### New platform analysis (`platforms/`)

```markdown
# [SoC Name] — Security Architecture Analysis

## SoC Overview
[brief chip description — CPU cores, bus architecture]

## Mechanism Inventory
[table: mechanism name → framework role → coverage]

## DMA Master Coverage
[list of non-CPU masters and their protection mechanism]

## Lock Sequence
[boot-time security configuration and lock sequence]

## Gaps and Limitations
[honest analysis of what the architecture cannot claim]

## References
```

### New use case (`use-cases/`)

```markdown
# [Use Case Title]

## Problem Statement
[what safety/security requirement is being addressed]

## Architecture
[which mechanisms are used and why]

## The Safety Argument
[how spatial FFI and temporal FFI are satisfied]

## The Security Argument
[what threat model is addressed]

## Certification Evidence
[what standards-normative claims this architecture supports]

## References
```

---

## Pull Request Process

1. Fork the repository and create a feature branch: `git checkout -b mechanism/risc-v-iopmp`
2. Follow the file structure templates above
3. Verify all claims have public source citations
4. Add your contribution to the relevant section of `README.md` (mechanisms table, platforms, or wanted list)
5. Open a PR with a description of what the contribution adds and the primary sources used
6. Be responsive to review comments — technical accuracy is more important than speed of merge

---

## Discussions

Use **GitHub Discussions** (enabled in this repo) for:
- "How does X map to the framework?" questions
- Requests for specific mechanisms or platforms
- Discussion of ambiguous certification claims
- General architecture questions

Discussions that reach a stable conclusion may be incorporated into the library as formal content.

---

## Code of Conduct

- Be precise and cite sources
- Acknowledge uncertainty honestly
- Engage with corrections constructively — the goal is accuracy
- Treat contributors from all backgrounds and disciplines with respect

Engineering communities that combine security and safety expertise are rare and valuable. Let's build one here.

---

## License

By contributing, you agree that your contributions will be licensed under [CC BY 4.0](LICENSE) — the same license as the rest of the repository. You retain copyright; the license grants anyone the right to use, share, and adapt your contribution with attribution.
