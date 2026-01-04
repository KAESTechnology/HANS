# Pull Request

Thank you for contributing to **HANS — Hardware-Aware Neural Storage**.

Please complete this template to help ensure high-quality, aligned contributions.

---

## Summary

Briefly describe what this PR changes and why.

---

## Motivation

Why is this change needed?

- What problem does it solve?
- How does it improve HANS?
- Is it addressing a bug, performance issue, or design gap?

---

## Type of Change

Check all that apply:

- [ ] Bug fix
- [ ] Performance improvement
- [ ] New feature
- [ ] Refactor / cleanup
- [ ] Documentation
- [ ] Benchmark / tooling
- [ ] Other (describe):

---

## Relation to HANS Design

Please indicate which core areas this PR affects:

- [ ] Canonical workload (`docs/CANONICAL_WORKLOAD.md`)
- [ ] Policy Engine (`docs/POLICY_ENGINE.md`)
- [ ] GPU / NVIDIA integration
- [ ] Power / thermal behavior (edge)
- [ ] Format-aware optimization
- [ ] Cache / tier management
- [ ] I/O engine (io_uring, async I/O)
- [ ] Benchmarks
- [ ] Documentation only

If applicable, reference relevant design documents.

---

## Performance Impact

Does this change affect performance?

- [ ] No
- [ ] Yes (improves performance)
- [ ] Yes (may regress performance)

If yes, please provide:

- Benchmark or workload used:
- Hardware configuration:
- Before / after metrics (if available):

Performance-related changes without measurements may be rejected.

---

## Edge & Safety Considerations

If applicable, describe how this change behaves under:

- Low system memory:
- Low VRAM:
- Power or thermal constraints:
- GPU errors or unavailable telemetry:

Safety and stability are prioritized over performance on edge devices.

---

## Testing

What testing was performed?

- [ ] Unit tests
- [ ] Integration tests
- [ ] Benchmarks
- [ ] Manual testing
- [ ] Not tested (explain why):

Include details or commands where relevant.

---

## Documentation

- [ ] No documentation changes needed
- [ ] Documentation updated
- [ ] New documentation added

If documentation was updated, list files:

---

## Checklist

Please confirm:

- [ ] This change aligns with the canonical workload
- [ ] Safety and fallback behavior are considered
- [ ] No unnecessary generalization was introduced
- [ ] Code is clear and commented where non-obvious
- [ ] Related documentation has been updated if needed

---

## Additional Notes

Anything else reviewers should know?
