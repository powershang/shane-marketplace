---
name: clock-reset-design
description: >-
  This skill handles clock and reset design including clock gating, reset
  synchronization, multi-clock domain design, clock dividers, reset trees,
  and power management. Use for "design clock gating", "reset synchronizer",
  "clock divider", "reset domain", "low power clock", or any clock/reset task.
---

# Clock and Reset Design

## Quick Mode
When user asks for quick clock/reset help, focus on these 2 checks only:
1. **Reset synchronization** — Verify async reset deassertion goes through a 2-FF synchronizer
2. **Clock gating** — Confirm ICG uses latch-based gating (no combinational AND gate on clock)

## When to Use
- Designing clock distribution and gating
- Implementing reset synchronization
- Creating clock dividers/multipliers
- Building reset trees for large designs
- Power management with clock gating

---

## Quick Reference

### Reset Types

| Type | Behavior | Pro | Con |
|------|----------|-----|-----|
| Async Assert, Sync Deassert | Most common | Clean deassert | Requires synchronizer |
| Fully Synchronous | Sync both | No metastability | May miss if clock stopped |
| Fully Asynchronous | Async both | Works without clock | Potential recovery issues |

### Clock Gating Styles

| Style | Power Savings | Complexity | Use Case |
|-------|---------------|------------|----------|
| AND gate | Low | Simple | Small blocks |
| Latch-based ICG | High | Medium | Standard approach |
| Clock mux | Medium | Medium | Mode switching |
| Fine-grain | Highest | High | Ultra-low power |

---

## Reset Synchronizer

### Standard Two-Flop Synchronizer

```systemverilog
module reset_sync #(
  parameter STAGES = 2  // Typically 2-3
)(
  input  logic clk,
  input  logic rst_n_async,  // Asynchronous reset input
  output logic rst_n_sync    // Synchronized reset output
);

  (* ASYNC_REG = "TRUE" *)
  logic [STAGES-1:0] sync_reg;

  always_ff @(posedge clk or negedge rst_n_async) begin
    if (!rst_n_async) begin
      sync_reg <= '0;
    end else begin
      sync_reg <= {sync_reg[STAGES-2:0], 1'b1};
    end
  end

  assign rst_n_sync = sync_reg[STAGES-1];

endmodule
```

### Reset Synchronizer with Glitch Filter

```systemverilog
module reset_sync_filtered #(
  parameter STAGES = 2,
  parameter FILTER_CYCLES = 4
)(
  input  logic clk,
  input  logic rst_n_async,
  output logic rst_n_sync
);

  (* ASYNC_REG = "TRUE" *)
  logic [STAGES-1:0] sync_reg;
  logic [$clog2(FILTER_CYCLES):0] filter_cnt;
  logic rst_n_filtered;

  // Synchronize
  always_ff @(posedge clk or negedge rst_n_async) begin
    if (!rst_n_async) begin
      sync_reg <= '0;
    end else begin
      sync_reg <= {sync_reg[STAGES-2:0], 1'b1};
    end
  end

  // Glitch filter - require N consecutive 1s
  always_ff @(posedge clk or negedge rst_n_async) begin
    if (!rst_n_async) begin
      filter_cnt <= '0;
      rst_n_filtered <= 1'b0;
    end else begin
      if (!sync_reg[STAGES-1]) begin
        filter_cnt <= '0;
        rst_n_filtered <= 1'b0;
      end else if (filter_cnt < FILTER_CYCLES) begin
        filter_cnt <= filter_cnt + 1;
      end else begin
        rst_n_filtered <= 1'b1;
      end
    end
  end

  assign rst_n_sync = rst_n_filtered;

endmodule
```

---

## Clock Gating

### Latch-Based Integrated Clock Gate (ICG)

```systemverilog
module clock_gate (
  input  logic clk,
  input  logic enable,
  input  logic test_en,    // Scan test mode bypass
  output logic gated_clk
);

  logic latch_out;

  // Negative-edge triggered latch
  // Enable captured when clock is low
  always_latch begin
    if (!clk) begin
      latch_out <= enable | test_en;
    end
  end

  // AND gate
  assign gated_clk = clk & latch_out;

endmodule
```

### Clock Gate with Power Management

```systemverilog
module clock_gate_pm #(
  parameter IDLE_CYCLES = 16  // Cycles before auto-gate
)(
  input  logic        clk,
  input  logic        rst_n,
  input  logic        sw_enable,      // Software enable
  input  logic        activity,       // Activity indicator
  input  logic        test_en,
  output logic        gated_clk,
  output logic        clock_stopped   // Status
);

  logic [$clog2(IDLE_CYCLES):0] idle_cnt;
  logic auto_gate;
  logic enable;

  // Idle counter
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      idle_cnt <= '0;
      auto_gate <= 1'b0;
    end else if (activity) begin
      idle_cnt <= '0;
      auto_gate <= 1'b0;
    end else if (idle_cnt < IDLE_CYCLES) begin
      idle_cnt <= idle_cnt + 1;
    end else begin
      auto_gate <= 1'b1;
    end
  end

  assign enable = sw_enable & ~auto_gate;

  clock_gate u_icg (
    .clk       (clk),
    .enable    (enable),
    .test_en   (test_en),
    .gated_clk (gated_clk)
  );

  assign clock_stopped = ~enable;

endmodule
```

---

## Clock Dividers

### Simple Divide-by-N (Power of 2)

```systemverilog
module clk_div_pow2 #(
  parameter DIV = 4  // Must be power of 2
)(
  input  logic clk,
  input  logic rst_n,
  output logic clk_div
);

  localparam CNT_W = $clog2(DIV);
  logic [CNT_W-1:0] cnt;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      cnt <= '0;
    end else begin
      cnt <= cnt + 1;
    end
  end

  assign clk_div = cnt[CNT_W-1];  // MSB toggles at DIV rate

endmodule
```

### Divide-by-N (Any Integer)

```systemverilog
module clk_div_n #(
  parameter DIV = 5  // Any integer >= 2
)(
  input  logic clk,
  input  logic rst_n,
  input  logic enable,
  output logic clk_div
);

  localparam CNT_W = $clog2(DIV);
  logic [CNT_W-1:0] cnt;
  logic clk_div_reg;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      cnt <= '0;
      clk_div_reg <= 1'b0;
    end else if (enable) begin
      if (cnt == DIV - 1) begin
        cnt <= '0;
        clk_div_reg <= ~clk_div_reg;
      end else begin
        cnt <= cnt + 1;
      end
    end
  end

  assign clk_div = clk_div_reg;

endmodule
```

### Fractional Clock Divider

```systemverilog
module clk_div_frac #(
  parameter WIDTH = 16
)(
  input  logic             clk,
  input  logic             rst_n,
  input  logic [WIDTH-1:0] div_int,   // Integer part
  input  logic [WIDTH-1:0] div_frac,  // Fractional part (Q0.16)
  output logic             clk_out
);

  logic [WIDTH:0]   cnt;
  logic [WIDTH-1:0] frac_acc;
  logic             carry;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      cnt <= '0;
      frac_acc <= '0;
      clk_out <= 1'b0;
    end else begin
      {carry, frac_acc} <= frac_acc + div_frac;

      if (cnt >= div_int + carry - 1) begin
        cnt <= '0;
        clk_out <= ~clk_out;
      end else begin
        cnt <= cnt + 1;
      end
    end
  end

endmodule
```

---

## Reset Tree

### Hierarchical Reset Distribution

```systemverilog
module reset_tree #(
  parameter NUM_DOMAINS = 4
)(
  input  logic             clk,
  input  logic             por_n,         // Power-on reset
  input  logic             sw_rst_n,      // Software reset
  input  logic [NUM_DOMAINS-1:0] domain_en,
  output logic [NUM_DOMAINS-1:0] domain_rst_n
);

  logic global_rst_n;
  logic global_rst_sync;

  // Combine reset sources
  assign global_rst_n = por_n & sw_rst_n;

  // Synchronize global reset
  reset_sync u_global_sync (
    .clk         (clk),
    .rst_n_async (global_rst_n),
    .rst_n_sync  (global_rst_sync)
  );

  // Per-domain reset synchronizers
  generate
    for (genvar i = 0; i < NUM_DOMAINS; i++) begin : gen_domain_rst
      logic domain_rst_async;

      assign domain_rst_async = global_rst_n & domain_en[i];

      reset_sync u_domain_sync (
        .clk         (clk),
        .rst_n_async (domain_rst_async),
        .rst_n_sync  (domain_rst_n[i])
      );
    end
  endgenerate

endmodule
```

### Reset Sequencer

```systemverilog
module reset_sequencer #(
  parameter NUM_STAGES = 4,
  parameter DELAY_CYCLES = 8
)(
  input  logic                    clk,
  input  logic                    rst_n_in,
  output logic [NUM_STAGES-1:0]   rst_n_out,  // Staged resets
  output logic                    seq_done
);

  logic [$clog2(DELAY_CYCLES):0] delay_cnt;
  logic [$clog2(NUM_STAGES):0]   stage;
  logic rst_n_sync;

  reset_sync u_sync (
    .clk         (clk),
    .rst_n_async (rst_n_in),
    .rst_n_sync  (rst_n_sync)
  );

  always_ff @(posedge clk or negedge rst_n_sync) begin
    if (!rst_n_sync) begin
      rst_n_out <= '0;
      delay_cnt <= '0;
      stage <= '0;
      seq_done <= 1'b0;
    end else begin
      if (stage < NUM_STAGES) begin
        if (delay_cnt < DELAY_CYCLES - 1) begin
          delay_cnt <= delay_cnt + 1;
        end else begin
          delay_cnt <= '0;
          rst_n_out[stage] <= 1'b1;
          stage <= stage + 1;
        end
      end else begin
        seq_done <= 1'b1;
      end
    end
  end

endmodule
```

---

## Multi-Clock Domain

### Clock Domain Crossing Reset

```systemverilog
module cdc_reset_bridge (
  input  logic clk_src,
  input  logic clk_dst,
  input  logic rst_n_src,    // Reset in source domain
  output logic rst_n_dst     // Reset in destination domain
);

  // Synchronize to destination domain
  // Note: Use longer chain for higher MTBF
  (* ASYNC_REG = "TRUE" *)
  logic [2:0] sync_reg;

  always_ff @(posedge clk_dst or negedge rst_n_src) begin
    if (!rst_n_src) begin
      sync_reg <= '0;
    end else begin
      sync_reg <= {sync_reg[1:0], 1'b1};
    end
  end

  assign rst_n_dst = sync_reg[2];

endmodule
```

### Glitch-Free Clock Mux

```systemverilog
module clk_mux_glitchfree (
  input  logic clk_a,
  input  logic clk_b,
  input  logic rst_n,
  input  logic sel,      // 0=clk_a, 1=clk_b
  output logic clk_out
);

  logic sel_a_sync, sel_b_sync;
  logic sel_a, sel_b;

  // Synchronize select to each clock domain
  // Then AND with opposite domain's deselect

  // Clock A domain
  always_ff @(posedge clk_a or negedge rst_n) begin
    if (!rst_n) begin
      sel_a_sync <= 1'b1;  // Default to clk_a
      sel_a <= 1'b1;
    end else begin
      sel_a_sync <= ~sel & ~sel_b;  // Select A when not B
      sel_a <= sel_a_sync;
    end
  end

  // Clock B domain
  always_ff @(posedge clk_b or negedge rst_n) begin
    if (!rst_n) begin
      sel_b_sync <= 1'b0;
      sel_b <= 1'b0;
    end else begin
      sel_b_sync <= sel & ~sel_a;  // Select B when not A
      sel_b <= sel_b_sync;
    end
  end

  // Output mux
  assign clk_out = (clk_a & sel_a) | (clk_b & sel_b);

endmodule
```

---

## Power Management

### Clock Request/Acknowledge

```systemverilog
module clock_req_ack (
  input  logic clk_free,       // Free-running clock
  input  logic rst_n,
  input  logic clk_req,        // Request clock
  output logic clk_ack,        // Clock available
  output logic clk_gated       // Gated clock output
);

  typedef enum logic [1:0] {
    CLK_OFF,
    CLK_STARTING,
    CLK_ON,
    CLK_STOPPING
  } state_t;

  state_t state;
  logic [3:0] settle_cnt;
  logic enable;

  always_ff @(posedge clk_free or negedge rst_n) begin
    if (!rst_n) begin
      state <= CLK_OFF;
      settle_cnt <= '0;
      enable <= 1'b0;
      clk_ack <= 1'b0;
    end else begin
      case (state)
        CLK_OFF: begin
          if (clk_req) begin
            enable <= 1'b1;
            state <= CLK_STARTING;
            settle_cnt <= '0;
          end
        end

        CLK_STARTING: begin
          if (settle_cnt < 4'd10) begin
            settle_cnt <= settle_cnt + 1;
          end else begin
            clk_ack <= 1'b1;
            state <= CLK_ON;
          end
        end

        CLK_ON: begin
          if (!clk_req) begin
            clk_ack <= 1'b0;
            state <= CLK_STOPPING;
            settle_cnt <= '0;
          end
        end

        CLK_STOPPING: begin
          if (settle_cnt < 4'd5) begin
            settle_cnt <= settle_cnt + 1;
          end else begin
            enable <= 1'b0;
            state <= CLK_OFF;
          end
        end
      endcase
    end
  end

  clock_gate u_icg (
    .clk       (clk_free),
    .enable    (enable),
    .test_en   (1'b0),
    .gated_clk (clk_gated)
  );

endmodule
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Reset deassertion metastability | Async reset deasserts near clock edge | Synchronize deassertion with 2-FF reset synchronizer |
| Glitchy clock gating | Combinational gate glitches when enable changes | Use latch-based ICG (enable captured when clk low) |
| Clock divider as clock source | Counter output creates timing analysis issues | Use clock enable pulse instead of generated clock |
| Reset release ordering | Downstream sees garbage before upstream is ready | Release upstream first, then downstream with sequencer |
| Missing test mode bypass | Clock gate blocks scan shift chain | OR `scan_enable` with functional enable in ICG |
| Async reset on data path | Reset glitch corrupts data registers | Use sync reset for data, async reset only for control/FSM |
| Clock domain crossing without sync | Metastability on signals crossing domains | Add proper 2-FF synchronizer or async FIFO at boundaries |

---

## Verification Checklist

- [ ] Reset synchronizers have correct number of stages
- [ ] Clock gating cells don't create glitches
- [ ] Reset release is synchronized to clock
- [ ] Clock dividers produce clean waveforms
- [ ] Multi-domain resets properly sequenced
- [ ] No combinational paths from async reset to data
- [ ] Test mode bypasses clock gates
- [ ] Power-up sequence correct

---

## References

- `references/reset-strategies.md` - Reset architecture patterns
- `references/clock-gating-styles.md` *(planned)* - Clock gating implementations
- `references/cdc-reset.md` *(planned)* - CDC for reset signals

## Examples

- `examples/example-clock-controller.md` - Complete clock controller
