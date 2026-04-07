# Always-Block Grouping Example: 4-Beat Collector

This document shows how to apply **Rule 4 (One always block, one signal group)** with the **trigger-set + semantic family** procedure.

## Example Module

```
module collector_4beat (clk, rst_n, valid_in, din, valid_out, dout, burst_cnt);
    parameter DW = 8;
    input              clk, rst_n, valid_in;
    input  [DW-1:0]    din;
    output reg         valid_out;
    output reg [DW*4-1:0] dout;
    output reg [3:0]   burst_cnt;
```

Behavior:
- IDLE → wait for `valid_in`
- COLLECT → consume 4 din beats
- On 4th beat → output 4-beat concat + burst_cnt+1 + return to IDLE

---

## ❌ Without Rule 4 (Monolithic)

```verilog
reg state;
reg [1:0] beat_cnt;
reg [DW-1:0] buf0, buf1, buf2, buf3;
localparam IDLE = 1'b0, COLLECT = 1'b1;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state     <= IDLE;
        beat_cnt  <= 0;
        buf0      <= 0; buf1 <= 0; buf2 <= 0; buf3 <= 0;
        valid_out <= 0;
        dout      <= 0;
        burst_cnt <= 0;
    end else begin
        valid_out <= 1'b0;
        if (state == IDLE) begin
            if (valid_in) begin
                state    <= COLLECT;
                beat_cnt <= 1;
                buf0     <= din;
            end
        end else begin // COLLECT
            if (valid_in) begin
                case (beat_cnt)
                    2'd1: buf1 <= din;
                    2'd2: buf2 <= din;
                    2'd3: buf3 <= din;
                endcase
                if (beat_cnt == 3) begin
                    state     <= IDLE;
                    beat_cnt  <= 0;
                    valid_out <= 1'b1;
                    dout      <= {din, buf2, buf1, buf0};
                    burst_cnt <= burst_cnt + 1;
                end else begin
                    beat_cnt <= beat_cnt + 1;
                end
            end
        end
    end
end
```

**Problem**: FSM, buffers, output, and counter are mixed in one 27-line block. Reading requires mentally separating different responsibilities.

---

## ✅ With Rule 4 (Trigger-Set Grouping)

### Step 1: List every reg with its write triggers

| Signal | Trigger conditions |
|--------|-------------------|
| `state`, `beat_cnt` | state==IDLE && valid_in / state==COLLECT && valid_in / done |
| `buf0` | state==IDLE && valid_in |
| `buf1..3` | state==COLLECT && valid_in @ beat_cnt=1..3 |
| `valid_out`, `dout`, `burst_cnt` | state==COLLECT && valid_in && beat_cnt==3 |

### Step 2: Group by trigger family + semantic family

| Group | Regs | Family |
|-------|------|--------|
| **FSM** | state, beat_cnt | FSM lifecycle (start/advance/done) |
| **Buffers** | buf0, buf1, buf2, buf3 | data capture family |
| **Output** | valid_out, dout, burst_cnt | output produce (done event) |

### Step 3: One group = one always block

```verilog
// =========================================================
// Block 1: FSM (state, beat_cnt)
// =========================================================
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state    <= IDLE;
        beat_cnt <= 0;
    end else if (state == IDLE) begin
        if (valid_in) begin
            state    <= COLLECT;
            beat_cnt <= 1;
        end
    end else begin // COLLECT
        if (valid_in) begin
            if (beat_cnt == 3) begin
                state    <= IDLE;
                beat_cnt <= 0;
            end else begin
                beat_cnt <= beat_cnt + 1;
            end
        end
    end
end

// =========================================================
// Block 2: Data buffers (buf0..3)
// =========================================================
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        buf0 <= 0; buf1 <= 0; buf2 <= 0; buf3 <= 0;
    end else begin
        if (state == IDLE && valid_in)
            buf0 <= din;
        if (state == COLLECT && valid_in) begin
            case (beat_cnt)
                2'd1: buf1 <= din;
                2'd2: buf2 <= din;
                2'd3: buf3 <= din;
            endcase
        end
    end
end

// =========================================================
// Block 3: Output + burst counter (valid_out, dout, burst_cnt)
// =========================================================
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        valid_out <= 0;
        dout      <= 0;
        burst_cnt <= 0;
    end else begin
        valid_out <= 1'b0;
        if (state == COLLECT && valid_in && beat_cnt == 3) begin
            valid_out <= 1'b1;
            dout      <= {din, buf2, buf1, buf0};
            burst_cnt <= burst_cnt + 1;
        end
    end
end
```

---

## Key Decision Point: Semantic > Trigger

Notice that `buf3` and `valid_out/dout/burst_cnt` share the **identical** trigger condition (`done`), yet they go in **different** groups.

**Why?**
- `buf3` belongs to "data capture" family (strongly coupled with buf0..2)
- `valid_out/dout/burst_cnt` belong to "output produce" family

**Rule of thumb**: When trigger overlap and semantic family conflict, **semantic family wins**.

---

## Comparison

| Item | Monolithic | Split (Rule 4) |
|------|-----------|---------------|
| Always blocks | 1 | 3 |
| Max block lines | 27 | 13 |
| Block responsibility | Mixed | Single (named) |
| Multi-driver risk | None (single block) | Must verify (Rule 4 guarantees separation) |
| Shared condition duplication | 0 | `state == COLLECT && valid_in && beat_cnt == 3` appears in Block 1 and 3 |
| Reading order | Top-to-bottom required | Jump to relevant block |

---

## Critical Anti-Pattern: Don't Use Shared Wires

❌ **Wrong** — using a shared wire to combine cross-block reg state:
```verilog
wire done = (state == COLLECT && valid_in && beat_cnt == 3);

always @(posedge clk) begin // Block 1: FSM
    if (done) state <= IDLE;
end

always @(posedge clk) begin // Block 3: Output
    if (done) valid_out <= 1'b1;
end
```

This **may** cause subtle simulation mismatches in edge cases (e.g. clk_in == clk_out scenarios), as we encountered in the descheduler refactor.

✅ **Right** — duplicate the condition in each block:
```verilog
always @(posedge clk) begin // Block 1: FSM
    if (state == COLLECT && valid_in && beat_cnt == 3)
        state <= IDLE;
end

always @(posedge clk) begin // Block 3: Output
    if (state == COLLECT && valid_in && beat_cnt == 3)
        valid_out <= 1'b1;
end
```

NBA semantics guarantee that all blocks read the same pre-clock values of `state`, `valid_in`, `beat_cnt`, so the duplicated conditions evaluate identically.
