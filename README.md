# HANS — Hardware-Aware Neural Storage

HANS is an open-source, AI-native, cache-first storage system designed to
maximize performance for modern AI workloads by being deeply aware of hardware —
especially GPUs and accelerators.

HANS sits between AI compute and persistent storage, acting as an intelligent,
distributed cache optimized for training, inference, and feature pipelines.

---

## Why HANS?

Traditional file systems and storage layers are not designed for AI workloads:

- GPUs stall waiting for data
- Training throughput is limited by I/O
- Inference latency is unpredictable
- Edge devices fail due to VRAM pressure, not compute limits

HANS addresses these issues by making hardware state and AI access patterns
first-class inputs to storage decisions.

---

## Key Features

- **Hardware-aware caching**
  - GPU utilization, VRAM pressure, NUMA locality
- **Memory-first architecture**
  - RAM and NVMe prioritized over disk
- **Edge-friendly design**
  - Stable operation under tight VRAM and power constraints
- **Asynchronous I/O**
  - Built on modern Linux primitives (io_uring)
- **AI-aware prefetching**
  - Optimized for epochs, shards, and checkpoints
- **Observability-driven**
  - eBPF and hardware metrics inform decisions

---

## What HANS Is (and Is Not)

### HANS Is:
- A cache-first storage layer for AI workloads
- Optimized for GPUs and edge device constraints
- Designed for Ubuntu and Linux environments
- Open-source and extensible

### HANS Is Not:
- A general-purpose filesystem
- A durable storage replacement
- A database or object store
- A universal drop-in solution for all workloads

Persistence is delegated to underlying storage systems.

---

## Architecture Overview

 ```mermaid
graph TD
    subgraph AI ["**_AI Frameworks & Applications_**"]
        direction LR
        PT[PyTorch] --- TF[TensorFlow] --- TR[Triton] --- FS[Feature Stores]
    end

    subgraph Clients ["**_Client Interfaces_**"]
        direction LR
        POSIX[POSIX / FUSE] --- PY[Python API] --- NA[Native] --- S3[S3]
    end

    subgraph Core ["**Core Cache Engine (THIS DESIGN)**"]
        direction LR
        PL[[Placement]] --- PR[[Prefetch]] --- CH[[Chunking]] --- EV[[Eviction]] --- IO[[I/O]] ---  GPU[[GPU-aware]] --- FA[[Format-aware]]
    end

    subgraph Storage ["**_Storage Tiers & Backends_**"]
        direction LR
        RAM[(RAM)] --- NVMe[(NVMe)] ---  DISK[(SSD)] --- OBJ[(Object Store)] --- RFS[(Remote FS)]
    end

    AI --> Clients
    Clients --> Core
    Core --> Storage
```

<!-- (Hide Class Diagram, use mermaid again when we publish)
## Class Diagram (Core Engine)

+--------------------------------------------------+
|                 CacheManager                     |
|--------------------------------------------------|
| - metadataStore                                  |
| - tierManager                                    |
| - placementEngine                                |
| - prefetchEngine                                 |
| - evictionEngine                                 |
| - ioEngine                                       |
| - telemetryEngine                                |
|--------------------------------------------------|
| + read(fileId, offset, len)                      |
| + write(fileId, buffer)                          |
| + prefetch(fileId, pattern)                      |
| + registerJob(jobProfile)                        |
+--------------------------------------------------+

        |                    |                    |
        v                    v                    v

+---------------+   +-------------------+   +-------------------+
| MetadataStore |   | TierManager       |   | IOEngine          |
|---------------|   |-------------------|   |-------------------|
| file -> chunks|   | RAM / NVMe / SSD  |   | io_uring backend  |
| access stats  |   | allocate/free     |   | async reads/writes|
+---------------+   +-------------------+   +-------------------+

        |                    |
        v                    v

+-------------------+   +-------------------+
| PlacementEngine   |   | EvictionEngine    |
|-------------------|   |-------------------|
| GPU-aware policy  |   | LRU / LFU / AI    |
| job locality      |   | priority-based   |
+-------------------+   +-------------------+

        |
        v

+-------------------+
| PrefetchEngine    |
|-------------------|
| pattern detection |
| async prefetch    |
+-------------------+

        |
        v

+-------------------+
| TelemetryEngine   |
|-------------------|
| eBPF signals      |
| metrics + tracing |
+-------------------+

For more details, see:
- `docs/VISION.md`
- `docs/ARCHITECTURE.md`
- `docs/EDGE_DESIGN.md`

---
-->

## Current Status

🚧 **Early development / pre-alpha**

HANS is currently under active development and not yet production-ready.

Initial focus:
- Core cache engine
- io_uring-based I/O
- GPU awareness
- Edge-friendly behavior

---

## Roadmap (High-Level)

- [ ] Core cache engine MVP
- [ ] io_uring integration
- [ ] RAM + NVMe tiering
- [ ] GPU-aware placement
- [ ] Edge profile support
- [ ] Benchmarks vs ext4 / XFS
- [ ] Public alpha release

---

## Getting Started

> ⚠️ Instructions will evolve as the project stabilizes.

### Requirements
- Ubuntu 22.04+
- Linux kernel with io_uring support
- NVIDIA GPU (recommended)
- CUDA toolkit (optional but recommended)

### Build (Early)

```bash
git clone https://github.com/ KAESTechnology/HANS.git
cd hans
# build instructions coming soon
