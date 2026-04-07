---
name: rtl-code-gen
description: >-
  This skill handles requests to "generate RTL", "write Verilog module",
  "generate FIFO", "write arbiter", "create AXI interface", "generate FSM",
  "write clock divider", "create dual-port RAM", "generate pipeline",
  "write RTL template", "scaffold Verilog module", or any RTL/hardware
  code generation task. Generates Verilog-2001 only (no SystemVerilog).
  Ensures generated code is synthesizable, lint-clean, and follows IC
  design best practices for ASIC and FPGA targets.
---

# RTL Code Generation

Generate synthesizable, lint-clean **Verilog-2001** code following strict coding rules for ASIC/FPGA targets.

## Quick Mode

When user asks for a quick generation check, focus on:
1. **Synthesizability** — No `initial`, `#delay`, `force/release`, system tasks
2. **Latch inference** — Every `if` has `else`, every `case` has `default`
3. **No SystemVerilog** — No `logic`, `always_comb`, `always_ff`

---

# Mandatory Coding Rules

## Rule 1: Verilog-2001 Only

**Forbidden** SystemVerilog constructs:
- `logic` → use `reg` / `wire`
- `always_comb` → use `always @(*)`
- `always_ff` → use `always @(posedge clk or negedge rst_n)`
- `'0`, `'1` fill literals → use explicit width like `{WIDTH{1'b0}}` or `0`
- `int` → use `integer`
- struct, interface, modport, packed array → not allowed
- `priority`/`unique` case → not allowed
- `/`, `%` → use shift-and-add or LUT

## Rule 2: 3-Always FSM Methodology (Strict)

Every FSM **must** be implemented as exactly 3 always blocks:

### Block 1: Sequential State Register
```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) current_state <= S_IDLE;
    else        current_state <= next_state;
end
```
- STRICTLY use `<=`
- **Never** put complex logic here

### Block 2: Combinational Next-State Logic
```verilog
always @(*) begin
    case (current_state)
        S_IDLE: next_state = (start) ? S_RUN : S_IDLE;
        S_RUN:  next_state = (done)  ? S_IDLE : S_RUN;
        default: next_state = S_IDLE;   // ← MANDATORY
    endcase
end
```
- STRICTLY use `=`
- **MUST** include `default: next_state = S_IDLE;` to prevent latches

### Block 3: Registered Output Decoding
```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        out_valid <= 1'b0;
        out_data  <= 0;
    end else begin
        case (current_state)
            S_RUN:  begin out_valid <= 1'b1; out_data <= compute_result; end
            default: out_valid <= 1'b0;
        endcase
    end
end
```
- Every output signal MUST have explicit reset value
- STRICTLY use `<=`

## Rule 3: Strict Separation of Concerns

### No Combinational Logic Inside Sequential Blocks

**STRICTLY FORBIDDEN** in `always @(posedge clk...)`:
- Declaring temporary variables (`integer`, `reg`)
- Using blocking assignments (`=`) for step-by-step calculation

### Next-Value Routing Rule

If a value needs multi-step calculation within a cycle:
1. Compute it in a **separate `always @(*)`** using `=`
2. Route the result via wire/reg
3. Assign to DFF in the sequential block using `<=`

**Bad** (forbidden):
```verilog
always @(posedge clk) begin
    integer wr_tmp;          // ❌ no temp var in seq block
    wr_tmp = wr_ptr;         // ❌ no blocking in seq block
    wr_tmp = wr_tmp + 1;
    wr_ptr <= wr_tmp;
end
```

**Good**:
```verilog
always @(*) begin
    wr_next = wr_ptr + 1;    // ✅ comb calculation
end
always @(posedge clk) begin
    wr_ptr <= wr_next;       // ✅ pure NBA
end
```

## Rule 4: One Always Block, One Signal Group (Trigger-Set Grouping)

When refactoring or designing multi-reg modules, split always blocks by **trigger-set + semantic family**.

### Procedure

**Step 1: List every reg + its write triggers**

| Signal | Trigger conditions |
|--------|-------------------|
| `state`, `count` | new_window, advance, done |
| `buf0..3` | data capture events |
| `valid_out`, `dout` | done event |

**Step 2: Group regs by trigger family**

Two regs belong in the same group if:
- They share the same **lifecycle event family** (e.g. all "data capture", all "FSM transition"), OR
- They are **strongly coupled state** components (e.g. `state` + `sub_phase`)

Two regs go in **different** groups if:
- Their semantic families differ even when triggers overlap (semantic > trigger)

**Step 3: One group = one always block**

No two always blocks may write the same reg (no multi-driver).

### Critical: No shared cross-block wires

Each block **duplicates its own condition logic** by reading module-level inputs and other blocks' regs directly. **Never** route cross-block reg state through shared intermediate wires.

All blocks on the same clock edge read pre-clock values (NBA guarantee), so duplicated conditions evaluate identically.

**Why**: Shared wires combining cross-block reg state (e.g. `wire new_window = !collecting && valid_in`) can cause subtle simulation mismatches in edge cases (e.g. clk_in == clk_out). Duplicate conditions in each block to avoid this.

### Example

See `references/always-grouping-example.md` for a worked 4-beat collector example showing monolithic vs split layout.

## Rule 5: Minimalist Architecture

- **Minimize MUXes**: Replace large MUXes with SRAM addressing. Store data in SRAM and let the FSM switch the `sram_addr`.
- **Resource Sharing**: Time-multiplex ALUs instead of parallel instantiation.
- **Simplify Math**: No `/` or `%`. Use shift-and-add or LUTs.

## Rule 6: Port & Module Style

- **Non-ANSI port style** (mandatory): module header has port names only, direction/width declared in module body
- One module per file, filename matches module name
- Parameters declared inside module body (`parameter`, not `parameter` in header)
- Consistent reset: async active-low (`negedge rst_n`)
- Registered outputs by default
- Snake_case naming, `clk_` prefix, `rst_` prefix, `_n` suffix for active-low

```verilog
// ✅ CORRECT — non-ANSI
module my_module (clk, rst_n, data_in, valid, data_out, ready);
    parameter DATA_W = 8;

    input               clk;
    input               rst_n;
    input  [DATA_W-1:0] data_in;
    input               valid;
    output [DATA_W-1:0] data_out;
    output              ready;

    // ... body ...
endmodule
```

```verilog
// ❌ WRONG — ANSI (never generate)
module my_module #(parameter DATA_W = 8) (
    input  [DATA_W-1:0] data_in,
    output [DATA_W-1:0] data_out
);
```

---

# Optional: Compute-Efficient Validation

When you want **100% syntax accuracy** without LLM reasoning, offload to a Python validator script. **Recommended but not mandatory** for every generation.

## Protocol

1. **Draft** code to a temporary file
2. **Validate** with Python script that checks:
   - ❌ Forbidden Keywords: `logic`, `always_comb`, `always_ff`, `/`, `%`
   - ❌ FSM Integrity: exactly 3 `always` blocks per FSM; Block 2 has `default:` case
   - ❌ Sequential Block Violation: regex scan inside `always @(posedge...)` for `=` (excluding relational ops) or `integer` declaration → CRITICAL ERROR
   - ❌ Combinational Block Violation: scan `always @(*)` for `<=` (excluding relational `<=`) → CRITICAL ERROR
3. **Iterate**: if script errors, fix draft and re-run
4. **Final output**: only release code when script returns Exit Code 0

When using validator, include the validation log in the output (Step 2 of workflow below).

---

# Mandatory Output Workflow

When generating RTL, output in this exact sequence:

## Step 1: Architecture Strategy
Briefly explain the SRAM/DFF time-multiplexing approach, FSM structure, and key design decisions.

## Step 2: Verification Log
- If using Python validator: paste the validator output proving deterministic checks passed
- If not using validator: paste a manual self-check checklist (see below)

Manual self-check checklist:
- [ ] No SystemVerilog (`logic`, `always_comb`, `always_ff`, `'0`, `int`)
- [ ] No `/` or `%`
- [ ] FSM uses 3-always pattern (if applicable)
- [ ] All `always @(*)` use only `=`, all `always @(posedge ...)` use only `<=`
- [ ] No `integer` declarations or `=` calculations inside sequential blocks
- [ ] Every `if` has `else`, every `case` has `default`
- [ ] All registers have explicit reset
- [ ] No two always blocks write the same reg
- [ ] Non-ANSI port style

## Step 3: Verified Verilog-2001 RTL Code
Output the final, checked code in a `verilog` code block.

---

# Generation Methodology (4-Step Process)

### Step 1: Clarify Requirements
- **Function**: datapath / controller / interface / memory
- **Target**: ASIC or FPGA
- **Interface**: port list with widths, clock/reset convention
- **Timing**: combinational or registered outputs? pipeline depth?
- **Parameterization**: which dimensions configurable?

### Step 2: Select Architecture Pattern
Match to known patterns from `references/common-patterns.md`:

| Requirement | Pattern |
|-------------|---------|
| Verification / testbench | see `references/testbench-patterns.md` |
| Buffering / rate matching | Sync FIFO or Async FIFO |
| Shared resource | Round-robin / priority arbiter |
| Control sequencing | 3-always FSM |
| Bus interface | AXI / AHB / APB bridge |
| Throughput | Pipeline stages |
| Frequency division | Clock divider |
| Storage | Single/dual-port RAM, register file |

### Step 3: Design Always-Block Layout
Apply Rule 4 trigger-set grouping:
1. List every reg + its write triggers
2. Group by trigger family + semantic family
3. Plan one always block per group

### Step 4: Generate + Validate
Apply Rules 1-6, run self-check or Python validator, then output via Step 1-3 of Output Workflow.

---

# Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Latch inference | Missing else/default | Complete all branches in `always @(*)` |
| Mixed blocking/non-blocking | Sim/synth mismatch | Only `<=` in seq, only `=` in comb |
| Temp vars in seq block | Violates Rule 3 | Move calc to `always @(*)` |
| `/` or `%` in RTL | Huge area, fails synth | Shift-and-add or LUT |
| Multi-driven signal | Two blocks driving same reg | Apply Rule 4: one reg, one block |
| Missing reset | X in gate sim | Add async/sync reset to every reg |
| ANSI port style | Violates Rule 6 | Use non-ANSI (names in header, decls in body) |

---

# Reference Files

- **`references/common-patterns.md`** — Reusable patterns: FIFO, arbiter, FSM, pipeline, clock divider, RAM, AXI-Lite. Note: when using, convert any `logic`/SystemVerilog to Verilog-2001.
- **`references/testbench-patterns.md`** — Testbench templates and patterns.
- **`references/always-grouping-example.md`** — Worked 4-beat collector example showing Rule 4 application.
- **`examples/example-gen-output.md`** — Full generation example walking through the 4-step process.
