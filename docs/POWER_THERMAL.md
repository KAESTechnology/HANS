# Power and Thermal Management

This document describes how HANS — Hardware-Aware Neural Storage — manages power consumption and thermal behavior on edge devices.

Unlike datacenter systems, edge devices have strict power budgets and thermal constraints.
HANS treats power and temperature as first-class inputs to storage decisions.

---

## Why Power and Thermal Matter

Edge AI devices face constraints that datacenters do not:

### Power Constraints
- Battery-powered devices have finite energy
- Industrial systems have per-device power budgets
- Edge devices usually have configurable power modes (5W, 10W, 15W, 30W, etc.)
- Exceeding power limits triggers system shutdown

### Thermal Constraints
- Passive cooling (no fans) is common
- Sustained high temperatures trigger throttling
- Thermal throttling reduces GPU frequency → hurts inference latency
- Overheating can cause system crashes

### Storage Impact
- Aggressive prefetching increases CPU/memory power draw
- NVMe SSDs consume 2-5W during active I/O
- Frequent I/O prevents system idle states
- Cache thrashing increases DRAM refresh power

HANS must balance performance with power/thermal stability.

---

## Target Devices

HANS power/thermal management is designed for:

### NVIDIA Jetson Family
- **Jetson Nano**: 5W / 10W modes
- **Jetson Xavier NX**: 10W / 15W / 20W modes
- **Jetson AGX Xavier**: 10W / 15W / 30W / 50W modes
- **Jetson Orin Nano**: 7W / 15W <!--SUPER MAXN is 25W-->modes
- **Jetson AGX Orin**: 15W / 30W / 50W / 60W modes

### Other Edge Platforms
- Industrial PCs with passive cooling
- Battery-powered inference devices
- Embedded systems with thermal limits

Datacenter GPUs (A100, H100) do not use these features.

---

## Power Mode Detection

HANS detects the active power mode at startup and runtime.

### Jetson-Specific APIs

On Jetson devices, power mode is controlled by `nvpmodel`:

```bash
# Query current power mode
nvpmodel -q

# Example output:
# NV Power Mode: MODE_15W
```

HANS queries this via system call:

```cpp
PowerMode detectJetsonPowerMode() {
    FILE* pipe = popen("nvpmodel -q", "r");
    
    char buffer[256];
    while (fgets(buffer, sizeof(buffer), pipe)) {
        if (strstr(buffer, "MODE_")) {
            // Parse power mode (e.g., "MODE_15W")
            return parsePowerMode(buffer);
        }
    }
    
    pclose(pipe);
    return PowerMode::UNKNOWN;
}
```

### Power Mode Enum

```cpp
enum class PowerMode {
    UNKNOWN,
    MODE_5W,    // Jetson Nano
    MODE_10W,   // Nano, Xavier, Orin
    MODE_15W,   // Xavier, Orin
    MODE_20W,   // Xavier
    MODE_30W,   // Xavier, Orin
    MODE_50W,   // Xavier, Orin
    MODE_60W,   // Orin
    MAXN        // Maximum performance (no limit)
};
```

---

## Thermal Monitoring

HANS continuously monitors thermal state via NVML.

### Temperature Zones

NVIDIA GPUs have multiple thermal sensors:

```cpp
struct ThermalState {
    int gpuTemp;        // GPU die temperature
    int memoryTemp;     // VRAM temperature (if available)
    int boardTemp;      // PCB temperature
};

ThermalState queryThermalState(nvmlDevice_t device) {
    ThermalState state;
    
    nvmlDeviceGetTemperature(
        device, 
        NVML_TEMPERATURE_GPU, 
        &state.gpuTemp
    );
    
    // Memory temp not available on all devices
    nvmlReturn_t ret = nvmlDeviceGetTemperature(
        device,
        NVML_TEMPERATURE_MEMORY,
        &state.memoryTemp
    );
    
    if (ret != NVML_SUCCESS) {
        state.memoryTemp = -1;  // Not available
    }
    
    return state;
}
```

### Thermal Zones

HANS defines thermal zones:

| Zone | Temperature Range | Behavior |
|------|-------------------|----------|
| **NORMAL** | < 60°C | Full performance |
| **WARM** | 60-75°C | Reduce background I/O |
| **HOT** | 75-85°C | Disable prefetch, reduce chunk size |
| **CRITICAL** | > 85°C | Minimal activity, emergency throttle |

Thresholds vary by device (Jetson Nano throttles earlier than AGX Xavier).

### Throttling Detection

HANS checks if the GPU is thermally throttled:

```cpp
bool isThrottled(nvmlDevice_t device) {
    unsigned long long reasons;
    nvmlDeviceGetCurrentClocksThrottleReasons(device, &reasons);
    
    return (reasons & nvmlClocksThrottleReasonThermalSlowdown) != 0;
}
```

If throttled, HANS immediately reduces activity.

---

## Power-Aware Policies

HANS adapts caching behavior based on power mode.

### Prefetch Aggressiveness

| Power Mode | Prefetch Behavior |
|------------|-------------------|
| 5W | Disabled (on-demand only) |
| 10W | Shallow (next 1-2 chunks) |
| 15W | Moderate (next 3-5 chunks) |
| 30W+ | Aggressive (next 5-10 chunks) |

**Rationale**: Prefetching increases memory traffic and CPU usage. In low-power
modes, the performance gain does not justify the energy cost.

### Background Persistence

| Power Mode | Persistence Behavior |
|------------|---------------------|
| 5W | Deferred until idle or explicit flush |
| 10W | Batched every 30s |
| 15W | Batched every 10s |
| 30W+ | Continuous background writes |

**Rationale**: Writing to storage (especially NVMe) consumes power. In low-power
modes, HANS defers writes until the system is idle.

### Chunk Size Scaling

| Power Mode | Default Chunk Size |
|------------|-------------------|
| 5W | 64 KB |
| 10W | 256 KB |
| 15W | 1 MB |
| 30W+ | 4 MB |

**Rationale**: Smaller chunks reduce per-operation memory and CPU overhead,
trading throughput for efficiency.

---

## Thermal-Aware Policies

HANS reacts to thermal state in real-time.

### Thermal Backpressure

```cpp
void applyThermalBackpressure(ThermalZone zone) {
    switch (zone) {
        case ThermalZone::NORMAL:
            // No restrictions
            break;
            
        case ThermalZone::WARM:
            // Reduce background I/O by 50%
            prefetchEngine->setAggressiveness(0.5);
            break;
            
        case ThermalZone::HOT:
            // Disable prefetch entirely
            prefetchEngine->disable();
            
            // Reduce chunk size to minimize CPU work
            chunkSize = std::min(chunkSize, 256 * 1024);
            break;
            
        case ThermalZone::CRITICAL:
            // Emergency: minimal activity
            prefetchEngine->disable();
            backgroundPersistence->pause();
            
            // Sleep briefly to allow cooling
            std::this_thread::sleep_for(std::chrono::seconds(1));
            break;
    }
}
```

### Throttle Response

When GPU throttling is detected:

```cpp
void onGpuThrottle() {
    // Immediately stop all non-critical I/O
    prefetchEngine->cancelAll();
    
    // Reduce active I/O to minimum
    ioEngine->setMaxConcurrency(1);
    
    // Log event
    telemetry->logEvent("GPU throttled, reducing HANS activity");
}
```

---

## Dynamic Power Mode Switching

Some edge deployments dynamically switch power modes (e.g., inference → 15W,
idle → 5W).

HANS monitors power mode changes:

```cpp
class PowerModeMonitor {
    PowerMode currentMode;
    
    void run() {
        while (running) {
            PowerMode newMode = detectJetsonPowerMode();
            
            if (newMode != currentMode) {
                onPowerModeChange(currentMode, newMode);
                currentMode = newMode;
            }
            
            std::this_thread::sleep_for(std::chrono::seconds(5));
        }
    }
    
    void onPowerModeChange(PowerMode old, PowerMode new) {
        // Reconfigure policies
        reconfigurePrefetch(new);
        reconfigurePersistence(new);
        reconfigureChunkSize(new);
        
        telemetry->logEvent("Power mode changed", old, new);
    }
};
```

---

## NVMe Power Management

NVMe SSDs have power states (Autonomous Power State Transition, APST).

HANS cooperates with NVMe power management:

### Idle Detection

After a period of inactivity, HANS allows NVMe to enter low-power states:

```cpp
void onIdleDetected() {
    // Flush pending writes
    ioEngine->flush();
    
    // Allow NVMe to enter low-power state
    // (no explicit action needed, just stop I/O)
}
```

### Batching I/O

In low-power modes, HANS batches small I/O operations to reduce NVMe wake-ups:

```cpp
void submitIO(IORequest request) {
    if (powerMode == PowerMode::MODE_5W || 
        powerMode == PowerMode::MODE_10W) {
        // Add to batch
        ioBatch.push_back(request);
        
        // Submit batch when large enough
        if (ioBatch.size() >= BATCH_SIZE) {
            ioEngine->submitBatch(ioBatch);
            ioBatch.clear();
        }
    } else {
        // Submit immediately
        ioEngine->submit(request);
    }
}
```

---

## CPU Frequency Awareness

Some edge systems dynamically scale CPU frequency (DVFS).

HANS queries CPU frequency (when available):

```cpp
int getCurrentCpuFreqMHz() {
    // Read from sysfs
    std::ifstream freq("/sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq");
    int freqKHz;
    freq >> freqKHz;
    return freqKHz / 1000;
}
```

If CPU is throttled (low frequency), HANS reduces CPU-intensive operations:
- Disable compression/decompression
- Skip format parsing (treat as binary)
- Reduce metadata lookups

---

## Battery-Powered Devices

For battery-powered systems, HANS can query battery state:

```cpp
struct BatteryState {
    int percentRemaining;
    bool charging;
    int estimatedMinutesRemaining;
};

BatteryState queryBattery() {
    // Linux: /sys/class/power_supply/BAT0/
    // Read capacity, status, etc.
}
```

Battery-aware behavior:
- < 20% battery: Disable prefetch, defer writes
- < 10% battery: Minimal activity, foreground only
- Charging: Resume normal operation

---

## Telemetry Integration

All power/thermal events are logged:

```cpp
struct PowerThermalEvent {
    Timestamp timestamp;
    PowerMode powerMode;
    ThermalZone thermalZone;
    int gpuTemp;
    bool throttled;
    std::string action;  // "disabled_prefetch", etc.
};
```

These events feed into:
- Prometheus metrics
- eBPF tracing
- Local logs

---

## Configuration

Power/thermal behavior is configurable:

```yaml
# hans.yaml
power:
  monitor_interval_ms: 5000
  
  # Per-mode policies
  modes:
    5W:
      prefetch: disabled
      chunk_size_kb: 64
      background_writes: false
    15W:
      prefetch: moderate
      chunk_size_kb: 1024
      background_writes: true

thermal:
  thresholds:
    warm: 60
    hot: 75
    critical: 85
  
  throttle_response:
    disable_prefetch: true
    reduce_chunk_size: true
```

---

## Interaction with GPU Utilization

Power/thermal policies interact with GPU-aware caching:

### High GPU Utilization + High Temperature
- Prioritize GPU data access (critical path)
- Disable background I/O (non-critical)
- Reduce chunk size (lower CPU overhead)

### Low GPU Utilization + Normal Temperature
- Increase prefetch (GPU is idle, prepare data)
- Background persistence (GPU not busy)

### Low GPU Utilization + High Temperature
- Minimal activity (let system cool)
- On-demand reads only

This is a complex state machine managed by the `TelemetryEngine`.

---

## Debugging and Diagnostics

### Power Mode History

HANS records power mode transitions:

```bash
$ hans-cli telemetry power-history

2025-01-02 10:00:00  MODE_30W -> MODE_15W  (user triggered)
2025-01-02 10:15:00  MODE_15W -> MODE_10W  (battery < 30%)
2025-01-02 10:45:00  MODE_10W -> MODE_15W  (charging)
```

### Thermal Timeline

HANS exports thermal data:

```bash
$ hans-cli telemetry thermal-timeline

Time                 GPU Temp  Zone      Throttled  Action
2025-01-02 10:00:00  55°C      NORMAL    No         -
2025-01-02 10:15:00  72°C      WARM      No         Reduced prefetch
2025-01-02 10:20:00  80°C      HOT       Yes        Disabled prefetch
2025-01-02 10:25:00  68°C      WARM      No         Re-enabled prefetch
```

---

## Future Enhancements

Planned improvements:

### Predictive Thermal Management
- Learn device-specific thermal behavior
- Preemptively reduce activity before throttling

### Battery State of Charge (SoC) Integration
- Adjust policies based on remaining battery capacity
- Predictive behavior based on charge/discharge rate

### Multi-Device Coordination
- In multi-GPU edge systems, prioritize cooling for active GPUs
- Shift workloads to cooler devices

---

## Summary

Power and thermal management are not optional features for edge AI.

HANS treats power mode and temperature as first-class signals that continuously
shape caching, prefetching, and I/O behavior.

By adapting to device constraints in real-time, HANS enables:
- Stable inference under thermal pressure
- Extended battery life
- Avoidance of system crashes due to overheating

Edge deployments are where HANS truly differentiates itself from general-purpose
filesystems.
