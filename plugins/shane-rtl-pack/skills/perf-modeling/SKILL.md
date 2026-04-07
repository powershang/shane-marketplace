---
name: perf-modeling
description: >-
  This skill handles performance modeling tasks including bandwidth estimation,
  latency analysis, bottleneck identification, and roofline modeling. Use for
  "performance model", "bandwidth analysis", "latency estimation", "bottleneck",
  or any performance characterization task.
---

# Performance Modeling Skill

Create and validate performance models for hardware designs.

## Quick Mode
When user asks for quick performance help, focus on these 2 checks only:
1. **Bandwidth bottleneck** — Calculate peak vs. sustained bandwidth at each interface and find the limiter
2. **Latency budget** — Sum worst-case latency per stage and compare against target

---

## When to Use

- Early architecture exploration
- Bandwidth/latency estimation
- Bottleneck identification
- Design trade-off analysis
- Performance regression tracking

---

## Methodology

### Step 1: Define Metrics

**Key performance metrics:**
```
Throughput Metrics:
├── Bandwidth (GB/s, transactions/sec)
├── IPC (instructions per cycle)
└── Utilization (% of peak)

Latency Metrics:
├── Average latency (cycles, ns)
├── Tail latency (99th percentile)
└── Jitter (variance)

Efficiency Metrics:
├── Power efficiency (ops/watt)
├── Area efficiency (ops/mm²)
└── Memory efficiency (useful/total bandwidth)
```

### Step 2: Create Analytical Model

**Queuing theory basics:**
```
Little's Law: L = λ × W
  L = average items in system
  λ = arrival rate
  W = average wait time

Utilization: ρ = λ / μ
  ρ = utilization (0 to 1)
  λ = arrival rate
  μ = service rate

For M/M/1 queue:
  W = 1 / (μ - λ)
  Response time = Service time / (1 - ρ)
```

### Step 3: Build Cycle-Accurate Model

```systemverilog
// Performance counter module.
// Assumes completions are in-order; use transaction IDs for out-of-order systems.
module perf_counters (
    input  logic        clk,
    input  logic        rst_n,
    // Events to count
    input  logic        req_valid,
    input  logic        req_ready,
    input  logic        resp_valid,
    // Counter outputs
    output logic [63:0] total_requests,
    output logic [63:0] total_cycles,
    output logic [63:0] busy_cycles,
    output logic [63:0] total_latency
);
    // Request tracking for latency measurement
    logic [15:0] start_cycle_q [16];
    logic [3:0]  alloc_ptr, retire_ptr, outstanding_count;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            total_requests <= '0;
            total_cycles   <= '0;
            busy_cycles    <= '0;
            total_latency  <= '0;
            alloc_ptr      <= '0;
            retire_ptr     <= '0;
            outstanding_count <= '0;
        end else begin
            total_cycles <= total_cycles + 1;

            // Count requests
            if (req_valid && req_ready) begin
                total_requests <= total_requests + 1;
                start_cycle_q[alloc_ptr] <= total_cycles[15:0];
                alloc_ptr <= alloc_ptr + 1'b1;
                outstanding_count <= outstanding_count + 1;
            end

            // Measure latency when the oldest in-flight request completes.
            if (resp_valid) begin
                total_latency <= total_latency +
                    (total_cycles[15:0] - start_cycle_q[retire_ptr]);
                retire_ptr <= retire_ptr + 1'b1;
                outstanding_count <= outstanding_count - 1;
            end

            // Track busy cycles
            if (outstanding_count > 0)
                busy_cycles <= busy_cycles + 1;
        end
    end
endmodule
```

### Step 4: Validate Against RTL

Compare model predictions vs. RTL simulation:
```
Metric              Model       RTL        Error
--------------------------------------------------
Avg Latency         12.5 cyc    13.2 cyc   5.3%
Peak Bandwidth      8.0 GB/s    7.6 GB/s   5.0%
Utilization         78%         75%        4.0%
```

### Step 5: Analyze Bottlenecks

```
System diagram with bottleneck analysis:

CPU ──[BW: 10 GB/s]──▶ Cache ──[BW: 8 GB/s]──▶ Memory
                         │                        │
                    (bottleneck)              (not limiting)

Cache bandwidth limits system to 8 GB/s
Improving memory won't help until cache is upgraded
```

---

## Quick Reference

### Bandwidth Calculation

```
Bandwidth = Data_width × Frequency × Utilization

Example:
  64-bit bus @ 1 GHz, 80% utilization
  BW = 64 bits × 1 GHz × 0.8 = 51.2 Gbps = 6.4 GB/s
```

### Latency Components

```
Total Latency = Pipeline + Queueing + Memory + Return

Example memory read:
  Pipeline:  2 cycles (address decode)
  Queueing:  5 cycles (average wait)
  Memory:    10 cycles (DRAM access)
  Return:    2 cycles (data return)
  Total:     19 cycles
```

### Utilization vs. Latency

```
ρ (utilization)  |  Latency multiplier
-----------------+---------------------
     50%         |       2×
     75%         |       4×
     90%         |      10×
     95%         |      20×
     99%         |     100×

Higher utilization → exponentially higher latency
```

---

## Performance Patterns

### Pipeline Throughput

```systemverilog
// Measure pipeline throughput
logic [31:0] valid_cycles;
logic [31:0] total_cycles;

always_ff @(posedge clk) begin
    total_cycles <= total_cycles + 1;
    if (pipe_valid)
        valid_cycles <= valid_cycles + 1;
end

// Throughput = valid_cycles / total_cycles
// Should approach 1.0 for fully utilized pipeline
```

### Memory Bandwidth

```systemverilog
// Measure effective memory bandwidth
logic [63:0] bytes_transferred;
logic [31:0] measurement_cycles;

always_ff @(posedge clk) begin
    if (mem_valid && mem_ready)
        bytes_transferred <= bytes_transferred + BEAT_SIZE;
    measurement_cycles <= measurement_cycles + 1;
end

// Bandwidth (GB/s) = bytes_transferred / measurement_cycles / clk_period_ns
```

### Latency Histogram

```systemverilog
// Track latency distribution
logic [31:0] latency_histogram [32];  // 32 buckets

always_ff @(posedge clk) begin
    if (transaction_complete) begin
        logic [4:0] bucket;
        bucket = (latency > 31) ? 31 : latency[4:0];
        latency_histogram[bucket] <= latency_histogram[bucket] + 1;
    end
end
```

---

## Analysis Templates

### Roofline Model

```
            Performance
                 │
      ┌──────────┼────────────── Peak compute
      │          │              /
      │          │            /
      │          │          / ← Memory-bound region
      │          │        /
      │          │      /
      │          │    /
      │          │  /
      └──────────┴────────────── Operational intensity
           (ops/byte)

If below roofline: memory-bound
If at ceiling: compute-bound
```

### Bottleneck Analysis

```
Component      Capacity    Demand    Utilization   Status
─────────────────────────────────────────────────────────
CPU            10 GOPS     8 GOPS    80%          OK
L1 Cache       100 GB/s    90 GB/s   90%          Warning
L2 Cache       50 GB/s     50 GB/s   100%         BOTTLENECK
Memory         25 GB/s     15 GB/s   60%          OK
```

---

## Checklist

- [ ] Key metrics defined
- [ ] Analytical model created
- [ ] RTL counters implemented
- [ ] Model validated against RTL
- [ ] Bottlenecks identified
- [ ] Trade-offs analyzed
- [ ] Performance regression tests added

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Ignoring queuing effects | Underestimating latency at high utilization | Apply queuing theory (M/M/1) |
| Peak vs. sustained confusion | Over-promising performance | Report both peak and sustained metrics |
| Missing contention | Model assumes exclusive access | Include arbitration and sharing effects |
| Wrong traffic pattern | Model doesn't match real workload | Profile actual application behavior |
| Counter overflow | Metrics wrap around silently | Use 64-bit counters, check saturation |
| Averaging hides tail | 99th percentile much worse than average | Track latency histograms |
| Model-RTL divergence | Model becomes stale | Validate model against RTL regularly |

---

## Resources

- **References:**
  - `references/bandwidth-latency-calc.md` - BW formulas, AMAT, Little's Law, roofline model, common gotchas

- **Examples:**
  - `examples/example-cache-model.md` - 2-level cache AMAT analysis with RTL perf counters and validation
