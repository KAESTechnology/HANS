# NVIDIA Integration

This document describes how HANS — Hardware-Aware Neural Storage — integrates
with NVIDIA GPUs, drivers, and acceleration technologies.

NVIDIA integration is a core differentiator for HANS. By treating GPU state as
a first-class input to storage decisions, HANS minimizes GPU idle time and
maximizes training/inference throughput.

---

## Integration Philosophy

HANS does not abstract away the GPU.

Instead, HANS:
- Continuously monitors GPU state via NVML
- Allocates memory with explicit CUDA awareness
- Reacts to pressure signals in real-time
- Exploits NVIDIA-specific acceleration where available

GPU state influences caching, prefetching, eviction, and I/O scheduling.

---

## NVIDIA Technology Stack

HANS integrates with the following NVIDIA technologies:

### Core (Always Used)
- **CUDA Runtime API**: Memory allocation, transfers, events
- **NVIDIA Management Library (NVML)**: Real-time GPU telemetry

### Datacenter (Conditional)
- **GPUDirect Storage (GDS)**: Direct NVMe-to-GPU transfers
- **CUDA Graphs**: Static execution optimization (future)

### Edge (Conditional)
- **Tegra System APIs**: Power modes, thermal management
- **jetson_clocks / nvpmodel**: Dynamic power/frequency control

---

## CUDA Runtime Integration

### Memory Allocation Strategy

HANS uses explicit CUDA memory management, avoiding Unified Memory.

#### Memory Types

| Type | Use Case | Allocation API |
|------|----------|----------------|
| Pinned Host | Primary cache tier | `cudaHostAlloc()` |
| Device (VRAM) | Ephemeral staging | `cudaMalloc()` |
| Unified Memory | **Never used** | ❌ |

#### Pinned Memory Pool

HANS maintains a pool of pre-allocated pinned buffers:

```cpp
class PinnedMemoryPool {
    // Pre-allocate at startup
    void initialize(size_t totalBytes);
    
    // Fast allocation from pool
    PinnedBuffer* allocate(size_t bytes);
    
    // Return to pool (no actual free)
    void release(PinnedBuffer* buffer);
    
    // Monitor fragmentation
    FragmentationStats getStats();
};
```

**Key Benefits:**
- Avoids repeated `cudaHostAlloc()` overhead
- Reduces memory fragmentation
- Predictable allocation latency

#### VRAM Allocation Rules

VRAM is used **only** when:
1. Data is actively being consumed by a GPU kernel
2. Free VRAM > 2x chunk size (safety margin)
3. GPU utilization > threshold (e.g., 70%)
4. No thermal throttling events

Otherwise, data remains in pinned host memory.

---

## CUDA Transfers

### Asynchronous Transfers

All CPU ↔ GPU transfers use CUDA streams:

```cpp
cudaMemcpyAsync(
    devicePtr,
    pinnedHostPtr,
    chunkSize,
    cudaMemcpyHostToDevice,
    stream
);
```

HANS tracks in-flight transfers and avoids oversubscribing PCIe bandwidth.

### Transfer Scheduling

Transfers are scheduled based on:
- GPU kernel launch queue depth
- Available PCIe bandwidth
- Job priority
- Prefetch hints

High-priority inference requests preempt background prefetch transfers.

### CUDA Events for Completion

HANS registers CUDA events to detect transfer completion without blocking:

```cpp
cudaEventRecord(completionEvent, stream);

// Later, non-blocking check:
if (cudaEventQuery(completionEvent) == cudaSuccess) {
    // Transfer complete, release pinned buffer
}
```

---

## NVIDIA Management Library (NVML)

NVML provides real-time GPU telemetry without blocking CUDA operations.

### Monitored Metrics

HANS continuously queries:

| Metric | NVML API | Update Frequency |
|--------|----------|------------------|
| GPU Utilization | `nvmlDeviceGetUtilizationRates()` | 100ms |
| VRAM Usage | `nvmlDeviceGetMemoryInfo()` | 100ms |
| Temperature | `nvmlDeviceGetTemperature()` | 500ms |
| Power Draw | `nvmlDeviceGetPowerUsage()` | 500ms |
| Clock Throttling | `nvmlDeviceGetCurrentClocksThrottleReasons()` | 1s |
| PCIe Throughput | `nvmlDeviceGetPcieThroughput()` | 1s |

> [!NOTE]
> Update Frequency is a guesstimation for now.

### Telemetry Thread

NVML queries run in a dedicated thread to avoid blocking I/O:

```cpp
class NvmlTelemetryThread {
    void run() {
        while (running) {
            // Query all metrics
            GpuMetrics metrics = queryNvml();
            
            // Publish to telemetry engine
            telemetryEngine->publish(metrics);
            
            // Sleep until next interval
            std::this_thread::sleep_for(updateInterval);
        }
    }
};
```

### Memory Pressure Detection

HANS defines memory pressure levels:

```cpp
enum MemoryPressure {
    NONE,       // < 70% VRAM used
    LOW,        // 70-85% VRAM used
    MODERATE,   // 85-95% VRAM used
    HIGH,       // 95-98% VRAM used
    CRITICAL    // > 98% VRAM used
};
```

Pressure levels influence:
- Chunk size (reduced under pressure)
- Prefetch aggressiveness (disabled under HIGH/CRITICAL)
- Eviction urgency (immediate under CRITICAL)

---

## GPUDirect Storage (GDS)

### What Is GDS?

GPUDirect Storage allows direct data transfers between NVMe storage and GPU
VRAM, bypassing CPU memory entirely.

**Benefits:**
- Eliminates CPU bounce buffer
- Reduces PCIe traffic
- Lower latency for large transfers

**Limitations:**
- Requires specific NVMe controllers
- Only available on datacenter GPUs (A100, H100, etc.)
- Not available on Jetson or consumer RTX

### GDS Detection

HANS detects GDS availability at startup:

```cpp
bool isGdsAvailable() {
    // Check driver version
    if (cudaDriverVersion < MIN_GDS_VERSION) {
        return false;
    }
    
    // Check device capability
    CUdevice device;
    int gdsSupported;
    cuDeviceGetAttribute(
        &gdsSupported,
        CU_DEVICE_ATTRIBUTE_GPU_DIRECT_RDMA_SUPPORTED,
        device
    );
    
    return gdsSupported == 1;
}
```

### GDS Usage Policy

HANS uses GDS **only** when:
- Transfer size > 4 MB (overhead otherwise)
- VRAM pressure is LOW or NONE
- Target data will be consumed immediately

For small chunks or edge devices, HANS falls back to pinned memory transfers.

### GDS API Integration

When GDS is available:

```cpp
// Open file with GDS flag
int fd = open(path, O_RDONLY | O_DIRECT);

// Register file descriptor with CUDA
CUfileHandle_t fileHandle;
cuFileHandleRegister(&fileHandle, fd);

// Direct read to GPU
cuFileRead(
    fileHandle,
    devicePtr,
    size,
    offset,
    0  // flags
);
```

---

## Platform-Specific Considerations

### Datacenter GPUs (V100, A100, H100, etc.)

**Characteristics:**
- Large VRAM (40-80 GB)
- High PCIe bandwidth (PCIe 4.0/5.0 x16)
- GDS available
- Stable power/thermal

**HANS Behavior:**
- Larger chunk sizes (8-64 MB)
- Aggressive prefetching
- GDS for large sequential reads
- VRAM used as secondary cache tier

### Consumer GPUs (RTX 3000/4000/5000 series)

**Characteristics:**
- Moderate VRAM (8-24 GB)
- PCIe 4.0 x16
- No GDS
- Variable power limits

**HANS Behavior:**
- Medium chunk sizes (2-16 MB) <!--maybe smaller (2-8 MB)?-->
- Conservative VRAM usage
- Pinned memory primary tier
- Prefetch based on utilization

### Jetson Edge Devices (Nano, Xavier, Orin)

**Characteristics:**
- Tiny VRAM (shared memroy, 2-16 GB)
- No discrete PCIe (integrated GPU)
- No GDS
- Strict power/thermal limits

**HANS Behavior:**
- Micro-chunks (64 KB - 1 MB)
- VRAM avoided entirely
- Pinned memory only
- Power-aware prefetch throttling

See `docs/EDGE_DESIGN.md` and `docs/POWER_THERMAL.md` for details.

---

## Handling GPU Out-of-Memory (OOM)

### OOM Detection

HANS intercepts CUDA allocation failures:

```cpp
cudaError_t err = cudaMalloc(&ptr, size);

if (err == cudaErrorMemoryAllocation) {
    handleGpuOOM();
}
```

### OOM Response

On OOM, HANS:
1. Immediately cancels all pending VRAM allocations
2. Triggers emergency eviction of non-critical VRAM buffers
3. Falls back to pinned host memory
4. Logs event to telemetry
5. Reduces chunk size for next allocation

**Critical Rule:**
HANS never retries failed VRAM allocations. Instead, it adapts by avoiding VRAM for the remainder of the job.

---

## Multi-GPU Support

HANS supports multi-GPU systems via explicit device context management.

### Device Affinity

Each cache tier is associated with a specific GPU:

```cpp
struct TierConfig {
    int cudaDeviceId;  // -1 for CPU-only tiers
    // ...
};
```

### NUMA Awareness

On multi-socket systems, HANS prefers:
- Pinned memory allocated on the NUMA node closest to the target GPU
- NVMe reads scheduled on the same NUMA node

NUMA locality is queried via:

```cpp
int numaNode = -1;
cudaDeviceGetAttribute(
    &numaNode,
    cudaDevAttrHostNativeAtomicSupported,  // proxy for NUMA info
    deviceId
);
```

### Cross-GPU Transfers

HANS avoids cross-GPU transfers when possible. If required:
- Use peer-to-peer (P2P) if available: `cudaDeviceEnablePeerAccess()`
- Otherwise, stage through pinned host memory

---

## Future NVIDIA Technologies

### CUDA Graphs (Planned)

CUDA Graphs allow capturing and replaying entire GPU execution sequences.

HANS could exploit this by:
- Prefetching data aligned with graph replay boundaries
- Pre-allocating VRAM based on static graph analysis

This is **future work** and not currently implemented.

### NVIDIA Grace-Hopper (Planned)

Grace-Hopper superchips feature coherent CPU-GPU memory.

HANS will need to:
- Detect Grace-Hopper architecture
- Exploit coherent memory for zero-copy access
- Reconsider pinned memory strategy

---

## Debugging and Diagnostics

### CUDA Error Handling

HANS wraps all CUDA calls with error checking:

```cpp
#define CUDA_CHECK(call) \
    do { \
        cudaError_t err = call; \
        if (err != cudaSuccess) { \
            logCudaError(__FILE__, __LINE__, err); \
            handleCudaError(err); \
        } \
    } while(0)
```

### NVML Logging

All NVML metrics are logged to the telemetry engine and exportable via:
- Prometheus metrics
- JSON logs
- In-memory ring buffer

### CUDA Profiling Integration

HANS supports NVIDIA Nsight Systems markers:

```cpp
nvtxRangePush("HANS: Prefetch Chunk");
// ... prefetch logic ...
nvtxRangePop();
```

---

## Summary

> [!IMPORTANT]
> NVIDIA integration is not an optional add-on for HANS — it is the foundation of the entire design.

By treating GPU state as a real-time input to caching decisions, HANS minimizes GPU idle time and maximizes throughput for AI workloads.

Key integration points:
- NVML for continuous telemetry
- Pinned host memory as primary cache
- Ephemeral VRAM with strict admission control
- GDS for datacenter acceleration
- Platform-specific optimizations for edge vs datacenter
