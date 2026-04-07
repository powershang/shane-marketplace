---
name: datapath-optimization
description: >-
  This skill handles datapath optimization techniques including arithmetic
  optimization, resource sharing, operator strength reduction, parallel vs
  sequential tradeoffs, and area/timing optimization. Use for "optimize
  datapath", "reduce area", "improve timing", "share resources", "optimize
  multiplier", or any datapath optimization task.
---

# Datapath Optimization

## Quick Mode
When user asks for quick datapath help, focus on these 2 checks only:
1. **Critical path** — Identify the longest combinational path and suggest pipelining or restructuring
2. **Resource sharing** — Find duplicate operators (multipliers, adders) that can be time-shared

## When to Use
- Optimizing arithmetic operations (add, multiply, divide)
- Reducing area through resource sharing
- Improving timing on critical paths
- Trading off area vs. throughput vs. latency
- Implementing efficient comparison and selection logic

---

## Quick Reference

### Optimization Techniques

| Technique | Area Impact | Timing Impact | When to Use |
|-----------|-------------|---------------|-------------|
| Resource Sharing | Reduces | May worsen | Low throughput requirements |
| Pipelining | Increases | Improves | High frequency designs |
| Strength Reduction | Reduces | Improves | Constant operations |
| Parallelization | Increases | Improves | High throughput needs |
| Retiming | Neutral | Improves | Balance pipeline stages |
| Tree Structures | Slight increase | Improves | Multi-input operations |

### Common Strength Reductions

| Original | Optimized | Savings |
|----------|-----------|---------|
| `x * 2^n` | `x << n` | Multiplier → shifter |
| `x / 2^n` | `x >> n` | Divider → shifter |
| `x % 2^n` | `x & (2^n - 1)` | Modulo → AND |
| `x * 3` | `(x << 1) + x` | Multiplier → shift+add |
| `x * 5` | `(x << 2) + x` | Multiplier → shift+add |
| `x * 7` | `(x << 3) - x` | Multiplier → shift+sub |
| `x * 9` | `(x << 3) + x` | Multiplier → shift+add |

---

## Arithmetic Optimization

### Addition Trees

**Bad**: Sequential addition (O(n) delay)
```systemverilog
// Sequential - long critical path
wire [31:0] sum = a + b + c + d + e + f + g + h;
```

**Good**: Tree structure (O(log n) delay)
```systemverilog
// Tree structure - shorter critical path
wire [31:0] sum_01 = a + b;
wire [31:0] sum_23 = c + d;
wire [31:0] sum_45 = e + f;
wire [31:0] sum_67 = g + h;

wire [31:0] sum_0123 = sum_01 + sum_23;
wire [31:0] sum_4567 = sum_45 + sum_67;

wire [31:0] sum = sum_0123 + sum_4567;
```

### Carry-Save Addition

For multi-operand addition, use carry-save to delay carry propagation:

```systemverilog
module csa_3to2 #(parameter W = 32)(
  input  logic [W-1:0] a, b, c,
  output logic [W-1:0] sum,
  output logic [W-1:0] carry
);
  assign sum   = a ^ b ^ c;
  assign carry = {((a & b) | (b & c) | (a & c)) [W-2:0], 1'b0};
endmodule

// Use CSA tree for 4+ operand addition
module sum_4_operands #(parameter W = 32)(
  input  logic [W-1:0] a, b, c, d,
  output logic [W:0]   result
);
  wire [W-1:0] s1, c1, s2, c2;

  csa_3to2 #(W) csa1 (.a(a), .b(b), .c(c), .sum(s1), .carry(c1));
  csa_3to2 #(W) csa2 (.a(s1), .b(c1), .c(d), .sum(s2), .carry(c2));

  assign result = s2 + c2;  // Single final CPA
endmodule
```

### Constant Multiplication

Replace multipliers with shift-add for constants:

```systemverilog
// x * 10 = x * 8 + x * 2
wire [31:0] times_10 = (x << 3) + (x << 1);

// x * 15 = x * 16 - x
wire [31:0] times_15 = (x << 4) - x;

// x * 100 = x * 64 + x * 32 + x * 4
wire [31:0] times_100 = (x << 6) + (x << 5) + (x << 2);

// Parameterized constant multiplier
function automatic logic [47:0] mul_const(
  input logic [31:0] x,
  input logic [15:0] c
);
  logic [47:0] result = '0;
  for (int i = 0; i < 16; i++) begin
    if (c[i]) result = result + (48'(x) << i);
  end
  return result;
endfunction
```

---

## Resource Sharing

### Time-Multiplexed Operations

Share expensive resources when operations don't overlap:

```systemverilog
module shared_multiplier #(
  parameter W = 16
)(
  input  logic             clk,
  input  logic             rst_n,
  input  logic [1:0]       op_sel,  // Select operation
  input  logic [W-1:0]     a, b, c, d,
  output logic [2*W-1:0]   result,
  output logic             valid
);

  logic [W-1:0] mul_a, mul_b;
  logic [2*W-1:0] mul_result;

  // Multiplexed inputs
  always_comb begin
    case (op_sel)
      2'd0: begin mul_a = a; mul_b = b; end
      2'd1: begin mul_a = c; mul_b = d; end
      2'd2: begin mul_a = a; mul_b = c; end
      default: begin mul_a = '0; mul_b = '0; end
    endcase
  end

  // Single shared multiplier
  assign mul_result = mul_a * mul_b;

  // Pipeline result
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      result <= '0;
      valid <= 1'b0;
    end else begin
      result <= mul_result;
      valid <= 1'b1;
    end
  end

endmodule
```

### FSM-Based Resource Sharing

For complex operations requiring multiple cycles:

```systemverilog
module shared_alu #(
  parameter W = 32
)(
  input  logic             clk,
  input  logic             rst_n,
  input  logic             start,
  input  logic [2:0]       opcode,
  input  logic [W-1:0]     operand_a,
  input  logic [W-1:0]     operand_b,
  output logic [W-1:0]     result,
  output logic             done
);

  typedef enum logic [2:0] {
    IDLE,
    OP_ADD,
    OP_MUL_1,
    OP_MUL_2,
    OP_DIV,
    DONE
  } state_t;

  state_t state, next_state;
  logic [W-1:0] acc, temp;
  logic [4:0] count;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      state <= IDLE;
      acc <= '0;
      temp <= '0;
      count <= '0;
    end else begin
      state <= next_state;
      case (state)
        OP_ADD: acc <= operand_a + operand_b;
        OP_MUL_1: begin
          acc <= '0;
          temp <= operand_a;
          count <= W;
        end
        OP_MUL_2: begin
          acc <= temp[W-1] ? (acc << 1) + operand_b : (acc << 1);
          temp <= temp << 1;
          count <= count - 1;
        end
        default: ;
      endcase
    end
  end

  always_comb begin
    next_state = state;
    done = 1'b0;
    case (state)
      IDLE: if (start) next_state = (opcode == 3'd0) ? OP_ADD :
                                    (opcode == 3'd1) ? OP_MUL_1 : IDLE;
      OP_ADD: next_state = DONE;
      OP_MUL_1: next_state = OP_MUL_2;
      OP_MUL_2: next_state = (count == 0) ? DONE : OP_MUL_2;
      DONE: begin done = 1'b1; next_state = IDLE; end
    endcase
  end

  assign result = acc;

endmodule
```

---

## Comparison Optimization

### Parallel Comparison Tree

```systemverilog
// Find minimum of 8 values using tree
module min_8 #(parameter W = 16)(
  input  logic [W-1:0] in [8],
  output logic [W-1:0] min_val,
  output logic [2:0]   min_idx
);

  // Level 1: Compare pairs
  wire [W-1:0] min_01 = (in[0] < in[1]) ? in[0] : in[1];
  wire [W-1:0] min_23 = (in[2] < in[3]) ? in[2] : in[3];
  wire [W-1:0] min_45 = (in[4] < in[5]) ? in[4] : in[5];
  wire [W-1:0] min_67 = (in[6] < in[7]) ? in[6] : in[7];

  wire idx_01 = (in[0] < in[1]) ? 1'b0 : 1'b1;
  wire idx_23 = (in[2] < in[3]) ? 1'b0 : 1'b1;
  wire idx_45 = (in[4] < in[5]) ? 1'b0 : 1'b1;
  wire idx_67 = (in[6] < in[7]) ? 1'b0 : 1'b1;

  // Level 2
  wire [W-1:0] min_0123 = (min_01 < min_23) ? min_01 : min_23;
  wire [W-1:0] min_4567 = (min_45 < min_67) ? min_45 : min_67;

  wire [1:0] idx_0123 = (min_01 < min_23) ? {1'b0, idx_01} : {1'b1, idx_23};
  wire [1:0] idx_4567 = (min_45 < min_67) ? {1'b0, idx_45} : {1'b1, idx_67};

  // Level 3
  assign min_val = (min_0123 < min_4567) ? min_0123 : min_4567;
  assign min_idx = (min_0123 < min_4567) ? {1'b0, idx_0123} : {1'b1, idx_4567};

endmodule
```

### Range Comparison

Optimize range checks:

```systemverilog
// Inefficient: Two comparisons
wire in_range = (x >= LOW) && (x <= HIGH);

// Efficient: Single subtraction + unsigned comparison
wire [31:0] offset = x - LOW;
wire in_range = offset <= (HIGH - LOW);  // Works if x >= LOW
```

---

## Multiplexer Optimization

### Priority Encoding

```systemverilog
// Inefficient: Nested ternary
wire [7:0] result = sel[3] ? d3 :
                    sel[2] ? d2 :
                    sel[1] ? d1 :
                    sel[0] ? d0 : '0;

// Better: Case statement (synthesis optimizes)
always_comb begin
  case (1'b1)
    sel[3]: result = d3;
    sel[2]: result = d2;
    sel[1]: result = d1;
    sel[0]: result = d0;
    default: result = '0;
  endcase
end

// Best for one-hot: Bitwise OR
wire [7:0] result = ({8{sel[0]}} & d0) |
                    ({8{sel[1]}} & d1) |
                    ({8{sel[2]}} & d2) |
                    ({8{sel[3]}} & d3);
```

### Multi-Stage Mux for Large Fan-in

```systemverilog
// 16:1 mux as two-stage 4:1
module mux_16to1 #(parameter W = 32)(
  input  logic [W-1:0] in [16],
  input  logic [3:0]   sel,
  output logic [W-1:0] out
);

  logic [W-1:0] stage1 [4];

  // Stage 1: Four 4:1 muxes
  always_comb begin
    for (int i = 0; i < 4; i++) begin
      case (sel[1:0])
        2'd0: stage1[i] = in[i*4 + 0];
        2'd1: stage1[i] = in[i*4 + 1];
        2'd2: stage1[i] = in[i*4 + 2];
        2'd3: stage1[i] = in[i*4 + 3];
      endcase
    end
  end

  // Stage 2: One 4:1 mux
  always_comb begin
    case (sel[3:2])
      2'd0: out = stage1[0];
      2'd1: out = stage1[1];
      2'd2: out = stage1[2];
      2'd3: out = stage1[3];
    endcase
  end

endmodule
```

---

## Shifter Optimization

### Barrel Shifter

```systemverilog
module barrel_shifter #(
  parameter W = 32,
  parameter S = $clog2(W)
)(
  input  logic [W-1:0]   data_in,
  input  logic [S-1:0]   shift_amt,
  input  logic           shift_right,
  input  logic           arithmetic,
  output logic [W-1:0]   data_out
);

  logic [W-1:0] stage [S+1];
  logic fill_bit;

  assign fill_bit = arithmetic & shift_right & data_in[W-1];
  assign stage[0] = data_in;

  generate
    for (genvar i = 0; i < S; i++) begin : shift_stages
      always_comb begin
        if (shift_amt[i]) begin
          if (shift_right)
            stage[i+1] = {{(1<<i){fill_bit}}, stage[i][W-1:(1<<i)]};
          else
            stage[i+1] = {stage[i][W-1-(1<<i):0], {(1<<i){1'b0}}};
        end else begin
          stage[i+1] = stage[i];
        end
      end
    end
  endgenerate

  assign data_out = stage[S];

endmodule
```

---

## Pipelining for Timing

### Before: Combinational Path Too Long

```systemverilog
// Long combinational path
always_comb begin
  temp1 = a * b;           // Multiplier
  temp2 = temp1 + c;       // Adder
  temp3 = temp2 >> 4;      // Shifter
  result = (temp3 > d) ? temp3 : d;  // Compare + mux
end
```

### After: Pipelined

```systemverilog
// Stage 1: Multiply
always_ff @(posedge clk) begin
  pipe1_mul <= a * b;
  pipe1_c <= c;
  pipe1_d <= d;
end

// Stage 2: Add
always_ff @(posedge clk) begin
  pipe2_sum <= pipe1_mul + pipe1_c;
  pipe2_d <= pipe1_d;
end

// Stage 3: Shift + Compare
logic [31:0] shifted;
assign shifted = pipe2_sum >> 4;

always_ff @(posedge clk) begin
  result <= (shifted > pipe2_d) ? shifted : pipe2_d;
end
```

---

## Verification Checklist

- [ ] Verify functional equivalence after optimization
- [ ] Check timing meets requirements
- [ ] Verify area reduction achieved
- [ ] Test edge cases (overflow, underflow)
- [ ] Confirm power reduction if applicable
- [ ] Validate resource sharing doesn't cause conflicts

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Sequential addition chain | O(n) critical path delay | Use tree structure for O(log n) |
| Multiplier for constants | Unnecessary area | Use shift-add decomposition |
| Resource sharing conflicts | Functional errors | Add arbitration or time-slot FSM |
| Over-optimization | Functional mismatch | Verify equivalence after changes |
| Ignoring overflow | Silent data corruption | Add saturation or explicit check |
| Wrong signedness | Incorrect arithmetic results | Match signed/unsigned consistently |
| Large fan-in MUX | Timing bottleneck | Use multi-stage MUX tree |
| Premature pipelining | Wasted registers, latency | Profile timing first, then optimize |

---

## References

- `references/arithmetic-opt.md` - Arithmetic optimization patterns
- `references/mux-opt.md` *(planned)* - Multiplexer optimization
- `references/timing-opt.md` *(planned)* - Critical path optimization

## Examples

- `examples/example-mac-unit.md` - Optimized MAC unit
- `examples/example-sorter.md` *(planned)* - Parallel sorting network
