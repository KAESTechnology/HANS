---
name: Performance Regression
about: Report performance regressions or unexpected benchmark results
title: "[Performance] "
labels: performance
assignees: ""
---

## Summary

Describe the performance issue or regression.

---

## Benchmark Context

Which benchmark or workload exposed the issue?

- Benchmark name / script:
- Dataset size:
- Access pattern (sequential, random, epochs, etc.):
- Cache configuration (RAM / NVMe / VRAM usage):
- Chunk size:

---

## Baseline Comparison

What are you comparing against?

- [ ] ext4
- [ ] XFS
- [ ] Previous HANS commit
- [ ] Other (describe):

Provide numbers if available.

---

## Metrics Observed

Include relevant metrics:

- GPU utilization:
- Throughput:
- Latency (p50, p75, p95 or p99 if available):
- Cache hit rate:
- VRAM usage:

Attach logs or graphs if possible.

---

## Environment

- OS & version:
- Kernel:
- CPU:
- GPU:
- Storage:
- Power mode (if edge):

---

## Expected Outcome

What performance improvement or stability did you expect?

---

## Notes

Performance issues should be evaluated against
`docs/CANONICAL_WORKLOAD.md` and `docs/BENCHMARKS.md`.
