---
name: arithmetic-units
description: >-
  Design of arithmetic units including multipliers (array, Booth, Wallace tree),
  dividers (restoring, non-restoring, SRT), and floating-point units. Covers
  fixed-point arithmetic, IEEE 754 basics, and area/timing/power tradeoffs.
  Use for "multiplier design", "divider", "FPU", "fixed-point", or arithmetic tasks.
---

# Arithmetic Units Design

## Quick Mode
When user asks for quick arithmetic unit help, focus on these 2 checks only:
1. **Signed/unsigned consistency** — Verify all operands use matching signedness throughout the datapath
2. **Overflow/saturation** — Confirm result width is sufficient or saturation logic is in place

## When to Use
- Designing multipliers (combinational or pipelined)
- Implementing dividers (iterative or high-radix)
- Creating fixed-point arithmetic units
- Building floating-point adders/multipliers
- Optimizing area/timing/power for arithmetic operations

---

## Quick Reference

### Multiplier Architectures

| Architecture | Latency | Throughput | Area | Best For |
|--------------|---------|------------|------|----------|
| Array | 1 cycle (comb) | 1/cycle | Large | Small widths, low freq |
| Booth Radix-2 | N/2 cycles | 1/N cycles | Small | Area-constrained |
| Booth Radix-4 | N/4 cycles | 1/(N/2) cycles | Medium | Balanced |
| Wallace Tree | 1 cycle (comb) | 1/cycle | Medium | High speed |
| Pipelined | P cycles | 1/cycle | Large | High throughput |

### Divider Architectures

| Architecture | Latency | Complexity | Best For |
|--------------|---------|------------|----------|
| Restoring | N cycles | Low | Simple, educational |
| Non-restoring | N cycles | Low | Slightly faster |
| SRT Radix-2 | N cycles | Medium | Practical designs |
| SRT Radix-4 | N/2 cycles | High | High performance |
| Newton-Raphson | ~log(N) iters | High | FPU, high precision |

### Fixed-Point Formats

| Format | Range | Resolution | Example |
|--------|-------|------------|---------|
| Q1.15 | [-1, 1) | 2^-15 | Audio DSP |
| Q8.8 | [-128, 128) | 2^-8 | General purpose |
| Q16.16 | [-32768, 32768) | 2^-16 | Graphics |
| Q1.31 | [-1, 1) | 2^-31 | High precision audio |

---

## Methodology

### Step 1: Requirements Analysis

**Key questions:**
1. Operand width? (8, 16, 32, 64 bits)
2. Signed or unsigned?
3. Latency requirement? (cycles)
4. Throughput requirement? (ops/cycle)
5. Target frequency?
6. Area budget?
7. Power constraints?

**Document specification:**
```
Arithmetic Unit: [multiplier/divider/FPU]
├── Operand width: [bits]
├── Result width: [bits]
├── Signed/Unsigned: [both/signed/unsigned]
├── Latency: [cycles]
├── Throughput: [ops/cycle]
├── Target freq: [MHz]
└── Special: [saturation, rounding mode, exceptions]
```

### Step 2: Architecture Selection

**Multiplier selection:**
```
                    ┌─────────────────┐
                    │ Throughput req? │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         1 op/cycle    1 op/N cycles   Variable
              │              │              │
              ▼              ▼              ▼
      ┌───────────┐   ┌───────────┐   ┌───────────┐
      │ Area      │   │ Iterative │   │ Variable  │
      │ budget?   │   │ Booth     │   │ latency   │
      └─────┬─────┘   └───────────┘   │ acceptable│
            │                         └───────────┘
     ┌──────┴──────┐
     ▼             ▼
  Tight         Relaxed
     │             │
     ▼             ▼
  Wallace      Pipelined
  Tree         Array
```

**Divider selection:**
```
Latency tolerance:
  - Must be 1-2 cycles → Use LUT/reciprocal + multiply
  - ~N cycles OK → SRT or Non-restoring
  - Iterative OK → Newton-Raphson (for FP)
```

---

## Multiplier Implementations

### Array Multiplier (Combinational)

Simple but large; O(N²) area:

```
         b3  b2  b1  b0
      ×  a3  a2  a1  a0
      ─────────────────
         a0b3 a0b2 a0b1 a0b0      ← Partial product 0
    a1b3 a1b2 a1b1 a1b0           ← Partial product 1
   a2b3 a2b2 a2b1 a2b0            ← Partial product 2
  a3b3 a3b2 a3b1 a3b0             ← Partial product 3
  ─────────────────────────────
   p7   p6   p5   p4   p3   p2   p1   p0
```

```systemverilog
module array_mult #(parameter W = 8)(
  input  logic [W-1:0] a, b,
  output logic [2*W-1:0] product
);
  logic [W-1:0] pp [W-1:0];  // Partial products

  // Generate partial products
  for (genvar i = 0; i < W; i++) begin : gen_pp
    assign pp[i] = b[i] ? a : '0;
  end

  // Sum partial products (shift and add)
  logic [2*W-1:0] sum [W-1:0];
  assign sum[0] = pp[0];

  for (genvar i = 1; i < W; i++) begin : gen_sum
    assign sum[i] = sum[i-1] + ({pp[i], {i{1'b0}}});
  end

  assign product = sum[W-1];
endmodule
```

### Booth Radix-4 Multiplier

Reduces partial products by half:

**Booth encoding table:**
| y[i+1] | y[i] | y[i-1] | Operation |
|--------|------|--------|-----------|
| 0 | 0 | 0 | +0 |
| 0 | 0 | 1 | +1×X |
| 0 | 1 | 0 | +1×X |
| 0 | 1 | 1 | +2×X |
| 1 | 0 | 0 | -2×X |
| 1 | 0 | 1 | -1×X |
| 1 | 1 | 0 | -1×X |
| 1 | 1 | 1 | +0 |

```systemverilog
module booth_radix4_mult #(parameter W = 16)(
  input  logic signed [W-1:0] multiplicand,
  input  logic signed [W-1:0] multiplier,
  output logic signed [2*W-1:0] product
);
  localparam NUM_PP = (W + 1) / 2;  // Number of partial products

  // Extended multiplier with implicit -1 bit
  logic signed [W:0] y_ext;
  assign y_ext = {multiplier, 1'b0};

  // Partial products
  logic signed [W+1:0] pp [NUM_PP-1:0];

  for (genvar i = 0; i < NUM_PP; i++) begin : gen_booth
    logic [2:0] booth_bits;
    assign booth_bits = y_ext[2*i+2 : 2*i];

    always_comb begin
      case (booth_bits)
        3'b000, 3'b111: pp[i] = '0;                              // +0
        3'b001, 3'b010: pp[i] = {{2{multiplicand[W-1]}}, multiplicand};  // +X
        3'b011:         pp[i] = {multiplicand[W-1], multiplicand, 1'b0}; // +2X
        3'b100:         pp[i] = -{multiplicand[W-1], multiplicand, 1'b0}; // -2X
        3'b101, 3'b110: pp[i] = -{{2{multiplicand[W-1]}}, multiplicand}; // -X
        default:        pp[i] = '0;
      endcase
    end
  end

  // Sum partial products with appropriate shifts
  logic signed [2*W-1:0] sum;
  always_comb begin
    sum = '0;
    for (int i = 0; i < NUM_PP; i++) begin
      sum = sum + (pp[i] <<< (2 * i));
    end
  end

  assign product = sum;
endmodule
```

### Wallace Tree Multiplier

Uses carry-save adders to reduce partial products in O(log N) stages:

```
Stage 0: 8 partial products
         ↓ (3:2 CSA reduction)
Stage 1: 6 terms
         ↓
Stage 2: 4 terms
         ↓
Stage 3: 3 terms
         ↓
Stage 4: 2 terms
         ↓
Final:   CPA (carry-propagate adder)
```

```systemverilog
// 3:2 Carry-Save Adder (Compressor)
module csa_3to2 #(parameter W = 32)(
  input  logic [W-1:0] a, b, c,
  output logic [W-1:0] sum,
  output logic [W-1:0] carry
);
  assign sum   = a ^ b ^ c;
  assign carry = (a & b) | (b & c) | (a & c);
endmodule

// 4:2 Compressor (two cascaded full adders)
module csa_4to2 #(parameter W = 32)(
  input  logic [W-1:0] a, b, c, d,
  input  logic [W-1:0] cin,
  output logic [W-1:0] sum,
  output logic [W-1:0] carry,
  output logic [W-1:0] cout
);
  // First level: 3:2 compress {a, b, c}
  logic [W-1:0] int_sum;
  assign int_sum = a ^ b ^ c;
  assign cout    = (a & b) | (b & c) | (a & c);  // majority(a,b,c)

  // Second level: 3:2 compress {int_sum, d, cin}
  assign sum   = int_sum ^ d ^ cin;
  assign carry = (int_sum & d) | (int_sum & cin) | (d & cin);  // majority(int_sum,d,cin)
endmodule
```

### Pipelined Multiplier

For high throughput at high frequencies:

```systemverilog
module pipelined_mult #(
  parameter W = 32,
  parameter STAGES = 4
)(
  input  logic             clk,
  input  logic             rst_n,
  input  logic [W-1:0]     a, b,
  input  logic             valid_in,
  output logic [2*W-1:0]   product,
  output logic             valid_out
);
  // Pipeline registers
  logic [2*W-1:0] pp_sum [STAGES:0];
  logic           valid_pipe [STAGES:0];

  // Divide partial products across stages
  localparam PP_PER_STAGE = (W + STAGES - 1) / STAGES;

  // Stage 0: Generate and sum first batch of partial products
  // Combinational partial-product accumulation
  logic [2*W-1:0] pp_acc_comb;
  always_comb begin
    pp_acc_comb = '0;
    for (int i = 0; i < PP_PER_STAGE && i < W; i++) begin
      if (b[i])
        pp_acc_comb = pp_acc_comb + ({{W{1'b0}}, a} << i);
    end
  end

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      pp_sum[0] <= '0;
      valid_pipe[0] <= 1'b0;
    end else begin
      valid_pipe[0] <= valid_in;
      pp_sum[0] <= pp_acc_comb;
    end
  end

  // Subsequent stages
  for (genvar s = 1; s <= STAGES; s++) begin : gen_stages
    always_ff @(posedge clk or negedge rst_n) begin
      if (!rst_n) begin
        pp_sum[s] <= '0;
        valid_pipe[s] <= 1'b0;
      end else begin
        valid_pipe[s] <= valid_pipe[s-1];
        pp_sum[s] <= pp_sum[s-1];
        // Add more partial products (simplified)
      end
    end
  end

  assign product = pp_sum[STAGES];
  assign valid_out = valid_pipe[STAGES];
endmodule
```

---

## Divider Implementations

### Non-Restoring Divider

```
Algorithm:
1. Initialize: R = dividend, D = divisor << N
2. For i = N-1 downto 0:
   a. If R >= 0: R = 2R - D, Q[i] = 1
   b. If R < 0:  R = 2R + D, Q[i] = 0
3. If R < 0: R = R + D (correction)
```

```systemverilog
module non_restoring_div #(parameter W = 32)(
  input  logic             clk,
  input  logic             rst_n,
  input  logic             start,
  input  logic [W-1:0]     dividend,
  input  logic [W-1:0]     divisor,
  output logic [W-1:0]     quotient,
  output logic [W-1:0]     remainder,
  output logic             done,
  output logic             div_by_zero
);
  typedef enum logic [1:0] {IDLE, COMPUTE, CORRECT, FINISH} state_t;
  state_t state;

  logic [W-1:0]   Q;           // Quotient accumulator
  logic [W:0]     R;           // Remainder (extra bit for sign)
  logic [W-1:0]   D;           // Divisor
  logic [$clog2(W)-1:0] count;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      state <= IDLE;
      done <= 1'b0;
      div_by_zero <= 1'b0;
    end else begin
      case (state)
        IDLE: begin
          done <= 1'b0;
          if (start) begin
            if (divisor == '0) begin
              div_by_zero <= 1'b1;
              done <= 1'b1;
            end else begin
              R <= {1'b0, dividend};
              D <= divisor;
              Q <= '0;
              count <= W - 1;
              state <= COMPUTE;
              div_by_zero <= 1'b0;
            end
          end
        end

        COMPUTE: begin
          // Shift and subtract/add
          if (!R[W]) begin
            // R positive: shift left, subtract D
            R <= {R[W-1:0], 1'b0} - {1'b0, D};
            Q <= {Q[W-2:0], 1'b1};
          end else begin
            // R negative: shift left, add D
            R <= {R[W-1:0], 1'b0} + {1'b0, D};
            Q <= {Q[W-2:0], 1'b0};
          end

          if (count == 0)
            state <= CORRECT;
          else
            count <= count - 1;
        end

        CORRECT: begin
          // Final correction if remainder negative
          if (R[W]) begin
            R <= R + {1'b0, D};
            Q <= Q - 1;
          end
          state <= FINISH;
        end

        FINISH: begin
          quotient <= Q;
          remainder <= R[W-1:0];
          done <= 1'b1;
          state <= IDLE;
        end
      endcase
    end
  end
endmodule
```

### SRT Radix-2 Divider

More efficient quotient digit selection:

```systemverilog
// SRT quotient digit selection (-1, 0, +1)
function automatic logic signed [1:0] srt_select(
  input logic [3:0] r_top,  // Top 4 bits of remainder
  input logic [2:0] d_top   // Top 3 bits of divisor
);
  // Simplified selection based on remainder estimate
  if ($signed(r_top) >= $signed({1'b0, d_top}))
    return 2'b01;   // +1
  else if ($signed(r_top) <= -$signed({1'b0, d_top}))
    return 2'b11;   // -1
  else
    return 2'b00;   // 0
endfunction
```

---

## Fixed-Point Arithmetic

### Q Format Notation

```
Qm.n format:
- m = integer bits (including sign for signed)
- n = fractional bits
- Total bits = m + n

Example: Q3.12 (16-bit signed)
  Bit 15: sign
  Bits 14-12: integer part
  Bits 11-0: fractional part

Value = (raw_value) × 2^(-n)
```

### Fixed-Point Multiplication

```systemverilog
module fixed_mult #(
  parameter Qi = 4,   // Input integer bits
  parameter Qf = 12,  // Input fractional bits
  parameter Wo = 16   // Output width
)(
  input  logic signed [Qi+Qf-1:0] a, b,
  output logic signed [Wo-1:0]    product
);
  localparam Wi = Qi + Qf;
  logic signed [2*Wi-1:0] full_product;

  assign full_product = a * b;

  // Result has 2*Qf fractional bits, need to shift right by Qf
  // and saturate/truncate to output width
  assign product = full_product[Qf +: Wo];
endmodule
```

### Saturation Logic

```systemverilog
function automatic logic signed [15:0] saturate_16(
  input logic signed [31:0] value
);
  if (value > 32'sh7FFF)
    return 16'sh7FFF;      // Positive saturation
  else if (value < -32'sh8000)
    return 16'sh8000;      // Negative saturation
  else
    return value[15:0];    // No saturation
endfunction
```

### Rounding Modes

```systemverilog
typedef enum logic [1:0] {
  ROUND_TRUNC,      // Truncate (toward zero)
  ROUND_NEAREST,    // Round to nearest, ties to even
  ROUND_UP,         // Round toward +infinity
  ROUND_DOWN        // Round toward -infinity
} round_mode_t;

// Synthesizable rounding: fixed frac_bits as parameter
function automatic logic [15:0] round_fixed #(parameter int FRAC = 8) (
  input logic [31:0] value,
  input round_mode_t mode
);
  logic [31:0] rounded;

  case (mode)
    ROUND_TRUNC:   rounded = value >> FRAC;
    ROUND_NEAREST: rounded = (value + (1 << (FRAC-1))) >> FRAC;
    ROUND_UP:      rounded = (value + ((1 << FRAC) - 1)) >> FRAC;
    ROUND_DOWN:    rounded = value >> FRAC;
  endcase

  return rounded[15:0];
endfunction
```

---

## Floating-Point Basics

### IEEE 754 Single Precision (32-bit)

```
┌───┬──────────┬───────────────────────┐
│ S │ Exponent │       Mantissa        │
│ 1 │    8     │          23           │
└───┴──────────┴───────────────────────┘

Value = (-1)^S × 1.Mantissa × 2^(Exponent - 127)

Special values:
- Zero:     E=0, M=0
- Denormal: E=0, M≠0, value = 0.M × 2^(-126)
- Infinity: E=255, M=0
- NaN:      E=255, M≠0
```

### FP Addition Algorithm

```
1. Unpack operands (sign, exponent, mantissa)
2. Align mantissas (shift smaller exponent's mantissa)
3. Add/subtract mantissas based on signs
4. Normalize result
5. Round
6. Handle special cases (overflow, underflow)
7. Pack result
```

```systemverilog
module fp_add (
  input  logic [31:0] a, b,
  input  logic [1:0]  rnd_mode,
  output logic [31:0] result,
  output logic [4:0]  flags  // Invalid, DivZero, Overflow, Underflow, Inexact
);
  // Unpack
  logic        a_sign, b_sign;
  logic [7:0]  a_exp, b_exp;
  logic [23:0] a_mant, b_mant;  // Implicit 1 included

  assign a_sign = a[31];
  assign a_exp  = a[30:23];
  assign a_mant = (a_exp == 0) ? {1'b0, a[22:0]} : {1'b1, a[22:0]};

  assign b_sign = b[31];
  assign b_exp  = b[30:23];
  assign b_mant = (b_exp == 0) ? {1'b0, b[22:0]} : {1'b1, b[22:0]};

  // Exponent difference and alignment
  logic [7:0] exp_diff;
  logic [7:0] result_exp;
  logic [26:0] a_aligned, b_aligned;  // Extra bits for guard, round, sticky

  // ... alignment and addition logic ...

endmodule
```

---

## Area/Timing/Power Tradeoffs

| Optimization | Area | Timing | Power | Technique |
|--------------|------|--------|-------|-----------|
| Reduce area | ↓ | ↑ latency | ↓ | Iterative, resource sharing |
| Improve timing | ↑ | ↓ | ↑ | Pipelining, parallel |
| Reduce power | ↑ | - | ↓ | Clock gating, operand isolation |

### Resource Sharing

```systemverilog
// Share one multiplier for multiple operations
module shared_mult #(parameter W = 16)(
  input  logic             clk,
  input  logic             sel,  // 0=op_a, 1=op_b
  input  logic [W-1:0]     a1, b1,  // Operands for op_a
  input  logic [W-1:0]     a2, b2,  // Operands for op_b
  output logic [2*W-1:0]   result
);
  logic [W-1:0] mux_a, mux_b;

  assign mux_a = sel ? a2 : a1;
  assign mux_b = sel ? b2 : b1;
  assign result = mux_a * mux_b;
endmodule
```

---

## Checklist

### Design Phase
- [ ] Define operand width and format (int/fixed/float)
- [ ] Specify latency and throughput requirements
- [ ] Choose architecture based on constraints
- [ ] Define rounding mode and saturation behavior
- [ ] Document exception handling (div-by-zero, overflow)

### Implementation Phase
- [ ] Implement core arithmetic logic
- [ ] Add pipeline registers if needed
- [ ] Implement special case handling
- [ ] Add saturation/rounding logic
- [ ] Include valid/ready handshaking

### Verification Phase
- [ ] Test corner cases (0, max, min, -1)
- [ ] Test overflow/underflow conditions
- [ ] Verify rounding modes
- [ ] Test special values (for FP)
- [ ] Compare against reference model

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Signed/unsigned mismatch | Wrong result for negative operands | Consistently use `signed` keyword |
| Missing sign extension | Truncated intermediate results | Extend to full width before operations |
| Overflow not handled | Silent wraparound corruption | Add saturation or exception logic |
| Combinational multiply timing | Critical path too long | Pipeline or use iterative architecture |
| Divide-by-zero crash | Simulation hangs or X output | Add explicit check and error flag |
| Rounding mode mismatch | Results differ from reference | Document and verify rounding mode |
| Fixed-point scaling error | Off by factor of 2^N | Track Q format through all operations |
| Booth encoding edge cases | Wrong result at boundaries | Test -1, 0, max, min exhaustively |

---

## Resources

- **References:**
  - `references/multiplier-architectures.md` - Detailed multiplier comparison
  - `references/divider-architectures.md` *(planned)* - Divider implementations
  - `references/fixed-point-guide.md` - Q format arithmetic
  - `references/floating-point-basics.md` *(planned)* - IEEE 754 guide

- **Examples:**
  - `examples/example-booth-mult.md` - Booth multiplier implementation
  - `examples/example-iterative-div.md` *(planned)* - Iterative divider

- **External:**
  - IEEE 754-2019 Standard
  - Computer Arithmetic: Algorithms and Hardware Designs (Parhami)
