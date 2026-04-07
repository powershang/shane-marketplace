---
name: pipeline-design
description: >-
  This skill handles pipeline architecture design including stage partitioning,
  hazard handling, backpressure propagation, skid buffers, and bubble management.
  Use for "design pipeline", "add pipeline stages", "implement backpressure",
  "create skid buffer", "handle pipeline stalls", or any pipelined datapath design.
---

# Pipeline Design

## Quick Mode
When user asks for quick pipeline help, focus on these 2 checks only:
1. **Handshake correctness** — Verify valid/ready protocol has no combinational loops
2. **Stall behavior** — Confirm data is held (not lost) when downstream stalls

## When to Use
- Designing multi-cycle datapaths that need throughput optimization
- Implementing flow control with backpressure
- Adding pipeline stages to meet timing closure
- Handling data hazards in processing pipelines

---

## Quick Reference

### Pipeline Fundamentals

```
Pipeline Throughput = 1 transaction / cycle (when full)
Pipeline Latency = N cycles (N = number of stages)
```

### Stage Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Combinational** | Logic only, no register | Small logic, timing OK |
| **Full Register** | Register on all outputs | Standard pipeline stage |
| **Skid Buffer** | 2-entry buffer for backpressure | AXI-style interfaces |
| **Bypass Stage** | Optional register (configurable) | Variable latency |

### Backpressure Patterns

| Pattern | Complexity | Bubble-free | Use Case |
|---------|------------|-------------|----------|
| Stall All | Low | No | Simple control |
| Credit-based | Medium | Yes | Known latency |
| Skid Buffer | Medium | Yes | AXI interfaces |
| Elastic Buffer | High | Yes | Variable latency |

---

## Methodology

### Step 1: Analyze Critical Path

```
1. Identify longest combinational path
2. Determine target frequency → max combinational delay
3. Calculate minimum stages needed:
   N_stages = ceil(critical_path_delay / target_cycle_time)
4. Add margin for routing: +10-20%
```

### Step 2: Partition into Stages

**Guidelines:**
- Balance delay across stages (±20% variation acceptable)
- Cut at natural boundaries (after MUX, after arithmetic)
- Keep related signals together (avoid inter-stage dependencies)
- Consider data width vs control width separately

**Common Partitioning:**

```
Stage 1: Decode / Setup
  - Address decode
  - Control signal generation
  - Operand selection

Stage 2: Execute / Compute
  - ALU operations
  - Memory address calculation
  - Data transformation

Stage 3: Writeback / Output
  - Result selection
  - Output formatting
  - Status generation
```

### Step 3: Design Flow Control

**Option A: Simple Valid Pipeline (No Backpressure)**

```systemverilog
// Valid propagates forward, no stalls
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    valid_s1 <= '0;
    valid_s2 <= '0;
  end else begin
    valid_s1 <= valid_in;
    valid_s2 <= valid_s1;
  end
end

// Data follows valid
always_ff @(posedge clk) begin
  if (valid_in) data_s1 <= data_in;
  if (valid_s1) data_s2 <= data_s1;
end
```

**Option B: Stall-based Pipeline (With Backpressure)**

```systemverilog
// Stall propagates backward
wire stall = !ready_out && valid_s2;

always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    valid_s1 <= '0;
    valid_s2 <= '0;
  end else if (!stall) begin
    valid_s1 <= valid_in;
    valid_s2 <= valid_s1;
  end
end

assign ready_in = !stall;
```

**Option C: Skid Buffer Pipeline (Bubble-free)**

```systemverilog
// See references/skid-buffer-patterns.md for full implementation
```

### Step 4: Handle Hazards

**Data Hazards:**
- RAW (Read After Write): Use forwarding/bypass
- WAW (Write After Write): Rare in simple pipelines
- WAR (Write After Read): Usually not an issue

**Control Hazards:**
- Branch/Jump: Flush pipeline or use speculation
- Exception: Precise exception requires careful ordering

**Structural Hazards:**
- Resource conflict: Add arbitration or duplicate resource

### Step 5: Implement Flush/Kill

```systemverilog
// Kill signal clears specific stages
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    valid_s1 <= '0;
  end else if (kill_s1) begin
    valid_s1 <= '0;  // Flush this stage
  end else if (!stall) begin
    valid_s1 <= valid_in;
  end
end
```

---

## Pipeline Register Template

```systemverilog
module pipe_stage #(
  parameter DATA_W = 32
)(
  input  logic             clk,
  input  logic             rst_n,
  // Upstream interface
  input  logic             valid_in,
  output logic             ready_in,
  input  logic [DATA_W-1:0] data_in,
  // Downstream interface
  output logic             valid_out,
  input  logic             ready_out,
  output logic [DATA_W-1:0] data_out,
  // Control
  input  logic             flush
);

  // Simple registered stage with backpressure
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      valid_out <= 1'b0;
    end else if (flush) begin
      valid_out <= 1'b0;
    end else if (ready_in) begin
      valid_out <= valid_in;
    end
  end

  always_ff @(posedge clk) begin
    if (ready_in && valid_in) begin
      data_out <= data_in;
    end
  end

  // Backpressure: accept new data if downstream ready OR no valid data held
  assign ready_in = ready_out || !valid_out;

endmodule
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Combinational stall loop | `stall` depends on `valid` which depends on `stall` | Break loop with register on `valid_out` |
| Data corruption on stall | Data overwritten when downstream not ready | Hold data: `if (ready && valid) data_out <= data_in` |
| Bubble insertion on stall | Stall-based pipeline creates idle cycles | Use skid buffer for bubble-free operation |
| Unbalanced stages | Bottleneck stage limits throughput | Balance timing across all stages within ±10% |
| Flush not clearing valid | Stale valid bits cause spurious processing | Reset all valid bits on flush signal |
| Forwarding path race | Forwarded data arrives late | Ensure forwarding MUX is in same pipeline stage |
| Missing reset on valid bits | Pipeline starts with X in valid registers | Reset all `valid` flip-flops to 0 |

---

## Verification Checklist

- [ ] Pipeline throughput = 1/cycle when not stalled
- [ ] No data loss during backpressure
- [ ] No data duplication during stalls
- [ ] Flush clears all target stages
- [ ] Valid/ready handshake correct (no combinational loops)
- [ ] Pipeline latency matches expected N cycles
- [ ] Forwarding paths work correctly (if applicable)
- [ ] Reset clears all valid bits

---

## References

- `references/pipeline-stages.md` - Stage partitioning strategies
- `references/skid-buffer-patterns.md` - Skid buffer implementations
- `references/backpressure-design.md` - Flow control patterns

## Examples

- `examples/example-axi-pipeline.md` - AXI read channel pipeline
