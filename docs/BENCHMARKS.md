# HANS Benchmarks

This document defines how performance is measured and evaluated for
HANS — Hardware-Aware Neural Storage.

Benchmarks exist to answer one question:

> Does HANS measurably improve AI workload performance compared to existing storage solutions?

If a benchmark does not help answer this question, it does not belong here.

---

## Benchmark Philosophy

HANS benchmarks are designed to:

- Measure **end-to-end AI impact**, not micro-optimizations in isolation
- Focus on **GPU utilization and data availability**
- Reflect **realistic AI access patterns**
- Be **reproducible and comparable**
- Favor **signal over synthetic perfection**

Benchmarks are tools for decision-making, not marketing.

---

## Canonical Benchmark Workload

All benchmarks are derived from the canonical workload defined in
`docs/CANONICAL_WORKLOAD.md`.

### Summary

- <!--NVIDIA--> GPU-based AI workload
- Repeated dataset access (multiple epochs)
- Mixed sequential and sharded reads
- Periodic checkpoint writes
- Ubuntu-based system
- Local or remote persistent storage

---

## Benchmark Categories

Benchmarks are grouped by intent.

---

### 1. Microbenchmarks (Early Development)

Purpose:
- Validate individual components
- Catch regressions early
- Compare implementation strategies

These do **not** prove HANS is useful on their own.

#### Examples
- Async read latency (io_uring vs POSIX)
- Chunk read/write throughput
- Metadata lookup latency
- Tier allocation overhead

#### Metrics
- Latency (p50, p75, p95 or p99)
- Throughput
- CPU overhead

---

### 2. Cache Effectiveness Benchmarks

Purpose:
- Measure the value of caching
- Validate placement and eviction policies

#### Scenario
- Repeated reads of the same dataset
- Cold cache vs warm cache
- RAM-only vs RAM + NVMe tiers

#### Metrics
- Cache hit rate
- Read latency
- Throughput improvement over baseline

---

### 3. End-to-End Training Benchmarks (Primary)

This is the **most important benchmark category**.

#### Scenario
- PyTorch training loop
- ImageNet-scale or equivalent dataset
- Multiple epochs
- Periodic checkpoint writes

#### Baselines
- ext4 on local SSD/NVMe
- XFS on local SSD/NVMe
- Direct object store access (if applicable)

#### Metrics
- GPU utilization (percentage (%))
- Time per epoch
- Time to first batch
- Total training time
- Checkpoint write latency

#### Success Criteria
- Reduced GPU idle time
- Faster epoch completion
- Stable performance across epochs

---

### 4. Inference Benchmarks

Purpose:
- Measure latency and stability under load
- Validate edge-friendly behavior

#### Scenario
- Triton or TensorRT inference server
- Fixed model
- Repeated inference requests
- Dataset accessed via HANS

#### Metrics
- P50, p75, P95 or 99 latency
- Throughput (requests/sec)
- VRAM usage stability
- Tail latency under memory pressure

---

### 5. Edge Device Benchmarks (Critical)

Edge benchmarks validate HANS’s core differentiator.

#### Scenario
- SBC <!--NVIDIA Jetson-class device-->
- Limited VRAM
- Continuous inference workload
- Background cache activity

#### Metrics
- GPU OOM events (must be zero)
- VRAM usage variance
- Inference latency stability
- Power consumption (optional)
- Thermal throttling events

#### Success Criteria
- No GPU crashes
- Predictable latency
- Graceful degradation under pressure

---

## Benchmark Environments

### Supported Environments

- Ubuntu 22.04+
- NVIDIA GPU <!--(RTX, Jetson, or datacenter)-->
- NVMe or SSD storage

### Required Reporting

Each benchmark run must report:
- OS and kernel version
- GPU model and driver version
- CPU model
- Storage type
- Dataset size
- Chunk size and cache configuration

Benchmarks without environment metadata are invalid.

---

## Benchmark Methodology

### Rules

- Always run **cold cache** and **warm cache**
- Discard first iteration when measuring steady-state
- Run multiple iterations and report averages
- Report variance where meaningful
- Never compare across different hardware without disclosure

---

## Visualization & Reporting

Recommended outputs:
- GPU utilization timelines
- Read latency histograms
- Cache hit rate over time
- VRAM usage over time

Graphs are preferred over single numbers.

---

## What We Do NOT Benchmark (By Design)

HANS explicitly does NOT benchmark:
- Metadata-heavy small file workloads
- POSIX edge-case compliance
- Transactional consistency
- Non-AI workloads

These are out of scope.

---

## Benchmark Evolution

Benchmarks will evolve as HANS matures.

Changes must be:
- Explicit
- Documented
- Motivated by real workloads

Historical benchmark results should be preserved for comparison.

---

## Final Note

Benchmarks are the truth.

If HANS does not outperform existing systems on these benchmarks, the design
must change — not the benchmarks.
