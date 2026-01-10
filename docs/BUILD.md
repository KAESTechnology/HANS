# Building HANS

This document describes how to build and run HANS — Hardware-Aware Neural Storage
— during its early development phase.

HANS is currently in **pre-alpha** and the build process is expected to evolve.
This document reflects the current intended workflow and constraints.

---

## Supported Platforms

HANS is currently being developed and will be tested on:

- **Ubuntu 22.04.5 LTS**
- Linux kernel with **io_uring** support
- x86_64 <!--Homelab-->
- ARM64 <!--(NVIDIA Jetson Orin Nano Super Developr Kit)-->

Other distributions may work but are not officially supported.

---

## Hardware Requirements

### Minimum
- 8 GB system RAM
- SSD or NVMe storage
- No GPU required (for basic development)

### Recommended (AI / Edge Development)
- NVIDIA GPU <!--(Jetson, RTX, or datacenter GPU)-->
- CUDA-capable driver
- NVMe SSD
- 16+ GB system RAM

---

## Software Dependencies

### Required
- Git
- Java
- CMake (>= 3.20)
- GCC / Clang
- Linux kernel headers

### Optional but Recommended
- NVIDIA CUDA Toolkit
- liburing
- eBPF tooling (bpftool, clang)
- Python 3.8+

---

## Repository Layout

<!--
# Proposed File & Module Architecture (HANS Core)
-->
```text
hans/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── BUG_REPORT.md
│   │   ├── config.yaml
│   │   ├── DESIGN.md
│   │   ├── PERFORMANCE.md
│   │   └── QUESTIONS.cpp
│   ├── CODEOWNERS
│   └── PULL_REQUEST_TEMPLATE.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── BENCHMARKS.md
│   ├── BUILD.md
│   ├── CANONICAL_WORKLOAD.md
│   ├── EDGE_DESIGN.md
│   ├── FORMAT_AWARE.md
│   ├── NVIDIA_INTEGRATION.md
│   ├── POLICY_ENGINE.md
│   ├── POWER_THERMAL.md
│   └── VISION.md
│
├── policy/
│   ├── PolicyContext.h
│   ├── PolicyDecision.h
│   ├── PolicyEngine.h
│   ├── PolicyEngine.cpp
│   ├── gpu/
│   │   ├── GpuPolicy.h
│   │   └── GpuPolicy.cpp
│   ├── power/
│   │   ├── PowerPolicy.h
│   │   └── PowerPolicy.cpp
│   ├── format/
│   │   ├── FormatPolicy.h
│   │   └── FormatPolicy.cpp
│   └── cache/
│       ├── CachePolicy.h
│       └── CachePolicy.cpp
│
├── shell/
│   ├── src/
│   │   ├── main/
│   │   └── test/
│   └── pom.xml
│
├── core/
│   ├── CacheManager.cpp
│   ├── CacheManager.h
│   ├── MetadataStore.cpp
│   ├── MetadataStore.h
│   └── JobContext.h
│
├── tiers/
│   ├── TierManager.h
│   ├── TierManager.cpp
│   ├── TierType.h
│   ├── RamTier.cpp
│   ├── NvmeTier.cpp
│   └── VramTier.cpp        # Ephemeral, optional
│
├── io/
│   ├── IOEngine.h
│   ├── IOEngine.cpp
│   ├── IoUringBackend.cpp
│   └── PosixFallback.cpp
│
├── gpu/
│   ├── GpuContext.cpp
│   ├── GpuMemoryManager.cpp
│   ├── CudaHooks.cpp
│   └── NvidiaMetrics.cpp
│
├── prefetch/
│   ├── PrefetchEngine.cpp
│   └── AccessPatternDetector.cpp
│
├── eviction/
│   ├── EvictionEngine.cpp
│   ├── policies/
│       ├── JobAwareLRU.cpp
│       └── GpuPressureAware.cpp
│
├── telemetry/
│   ├── TelemetryEngine.h
│   ├── TelemetryEngine.cpp
│   ├── TelemetrySnapshot.h
│   ├── EBPFCollector.cpp
│   └── MetricsExporter.cpp
│
├── util/
│   ├── Logger.h
│   └── Logger.cpp
│
├── benchmarks/
│   ├── micro/
│   ├── training/
│   └── edge/
|
├── client/
│   ├── FuseClient/
│   ├── Python/
│   └── Native/
│
├── tests/
│   ├── integration/
│   └── gpu/
│
├── build/
│   └── cmake/
│       └── CMakeLists.txt
│
├── LICENCE
├── NOTICE
├── CONTRIBUTING.md
└── README.md

```

---

## Build Philosophy

HANS favors:
- Explicit dependencies over auto-detection
- Debuggable builds over opaque optimizations
- Reproducibility over convenience

Early builds prioritize correctness and instrumentation.

---

## Build Steps

> ⚠️ These steps will change as HANS evolves.

### 1. Clone the Repository

```bash
git clone https://github.com/KAESTechnology/HANS.git
cd hans
