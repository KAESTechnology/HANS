# Security Advisories

This document describes how security advisories are handled for
**HANS — Hardware-Aware Neural Storage**.

HANS is an early-stage, performance-oriented system intended for trusted
environments. Nevertheless, security issues are taken seriously and handled
responsibly.

---

## Scope of Security Advisories

Security advisories for HANS may include issues related to:

- Crashes caused by malformed input or unexpected states
- Unsafe interactions with hardware (GPU, memory, I/O)
- Privilege misuse or unintended access
- Denial-of-service conditions under specific workloads
- Data corruption under error conditions

Advisories are **not limited to vulnerabilities with remote exploitability**.
Stability and safety issues may also warrant advisories.

---

## What Is Not a Security Advisory

The following are **not** considered security advisories for HANS:

- Performance regressions
- Benchmark inaccuracies
- Missing features
- Lack of hardening against adversarial workloads
- Behavior explicitly documented as out of scope in `SECURITY.md`

These should be reported via standard GitHub issues instead.

---

## Reporting a Security Issue

If you believe you have found a security issue in HANS:

1. **Do not open a public GitHub issue**
2. Report the issue privately to the maintainer

When reporting, please include:
- A clear description of the issue
- Affected versions or commits
- Steps to reproduce (if available)
- Potential impact and severity
- Any relevant logs or traces

A private reporting contact will be published once the project matures.

---

## Triage & Response Process

Reported security issues are handled as follows:

1. **Acknowledgement**
   - The report will be acknowledged when received.

2. **Assessment**
   - The issue is evaluated for severity, impact, and scope.
   - Reproducibility is confirmed where possible.

3. **Mitigation**
   - Fixes or workarounds are developed as appropriate.
   - Priority is based on safety and impact.

4. **Disclosure**
   - Public disclosure occurs after a fix or mitigation is available,
     when appropriate.

Timelines may vary due to the early-stage nature of the project.

---

## Advisory Publication

When a security advisory is published, it may include:

- Affected versions or commit ranges
- Description of the issue
- Impact and severity assessment
- Mitigation steps or fixes
- Upgrade or configuration recommendations

Advisories may be published as:
- GitHub Security Advisories
- Release notes
- Documentation updates

Not all issues will result in a formal advisory.

---

## Severity Classification (Informal)

HANS uses an informal severity classification:

- **Critical**  
  System crashes, GPU instability, or data corruption with minimal user action.

- **High**  
  Denial-of-service, unsafe hardware behavior, or privilege misuse under
  realistic conditions.

- **Medium**  
  Incorrect behavior or instability requiring unusual configurations.

- **Low**  
  Minor issues with limited impact or easy mitigation.

This classification is advisory, not contractual.

---

## Supported Versions

Given the early development stage:

- Only the **latest development branch** is supported
- Older commits are not patched retroactively
- Users are expected to upgrade frequently

Long-term support policies may be introduced later.

---

## Coordinated Disclosure

HANS follows coordinated disclosure principles:

- Reporters are encouraged to allow time for fixes
- Public disclosure should be coordinated with the maintainer
- Credit may be given to reporters where appropriate

HANS values responsible collaboration.

---

## Relationship to SECURITY.md

This document complements `SECURITY.md`.

- `SECURITY.md` defines **assumptions and scope**
- `SECURITY_ADVISORIES.md` defines **process and response**

Both should be read together.

---

## Summary

HANS treats safety and stability issues seriously, even when operating in
trusted environments.

By defining a clear advisory process early, HANS aims to maintain trust while
remaining honest about its scope and maturity.
