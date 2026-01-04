# Canonical Workload for HANS

This document defines the **canonical workload** used to guide the design,
implementation, and benchmarking of HANS — Hardware-Aware Neural Storage.

All architectural decisions must be justifiable against this workload.

---

## Purpose of a Canonical Workload

HANS is not a general-purpose filesystem.

Without a canonical workload:
- Performance claims are meaningless
- Optimizations become arbitrary
- Scope creep is inevitable

This workload defines:
- What HANS must do exceptionally well
- What trade-offs are acceptable
- What is explicitly out of scope

---

## Canonical Workload Definition

### Workload Type

**GPU-based AI training or inference pipeline with repeated dataset access**

---

## Primary Scenario (Baseline)

### Description

A single-node or small-cluster system running an AI workload that:

- Reads large datasets repeatedly (multiple epochs)
- Uses NVIDIA GPUs
- Suffers from I/O stalls and GPU underutilization
- Runs on Ubuntu
- Uses persistent storage (local or remote)

---

### Concrete Example (Baseline)

**Framework**
- PyTorch

**Hardware**
- NVIDIA GPU <!--(RTX / Jetson / Datacenter)-->
- Limited VRAM (especially on edge)
- NVMe-backed storage

**Dataset**
- ImageNet-scale dataset or equivalent
- Files sized from KBs to MBs
- Sequential and sharded access patterns

**Access Pattern**
- Repeated sequential scans
- Epoch-based reuse
- Occasional random access
- Periodic checkpoint writes

---

## Key Performance Questions

HANS must answer:

1. Can GPU idle time be reduced?
2. Can repeated dataset reads be accelerated?
3. Can VRAM pressure be controlled without crashes?
4. Can checkpoint writes avoid stalling compute?
5. Can performance remain stable under memory pressure?

If an optimization does not improve at least one of these, it is suspect.

---

## Metrics That Matter

### Primary Metrics
- GPU utilization (%)
- End-to-end training/inference throughput
- Data read latency
- Cache hit rate
- VRAM usage stability

### Secondary Metrics
- CPU utilization
- Power consumption (edge)
- Memory fragmentation
- Restart recovery time

---

## Baselines for Comparison

HANS should be compared against:

- ext4 on local SSD/NVMe
- XFS on local SSD/NVMe
- Direct object store access (if applicable)

Benchmarks must be reproducible.

---

## Success Criteria (Phased)

### Phase 1 (Early)
- Faster repeated reads than ext4/XFS
- No regressions vs others

### Phase 2 (GPU-Aware)
- Reduced GPU idle time
- Stable VRAM usage
- Faster time-to-first-batch

### Phase 3 (Edge)
- Stable inference under tight VRAM constraints
- No GPU OOM events
- Predictable latency

---

## Explicit Non-Goals

The canonical workload does NOT include:
- High-concurrency OLTP
- Small file metadata-heavy workloads
- POSIX compliance edge cases
- Strong consistency guarantees
- Non-AI workloads

Optimizations for these are out of scope unless they directly benefit the
canonical workload.

---

## How to Use This Document

Before implementing a feature, ask:
- Does this help the canonical workload?
- Which metric does it improve?
- Is the trade-off acceptable?

If the answer is unclear, defer the feature.

---

## Evolution of the Canonical Workload

The canonical workload may evolve, but changes must be:
- Explicit
- Documented
- Justified by real AI use cases

This document is a guardrail, not a constraint.
