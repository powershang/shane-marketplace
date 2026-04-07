---
name: eco-handling
description: >-
  Engineering Change Order (ECO) handling for post-synthesis and post-layout
  fixes. Covers functional ECOs, timing ECOs, spare cell usage, and verification
  impact analysis. Use for "ECO", "metal fix", "spare cell", "late change", or
  post-tapeout modification tasks.
---

# ECO Handling

## Quick Mode
When user asks for quick ECO help, focus on these 2 checks only:
1. **Impact scope** — List all affected signals, modules, and timing paths
2. **Spare cell availability** — Check if spare cells are sufficient and proximate for the fix

## When to Use
- Making late-stage functional fixes
- Fixing timing violations post-synthesis
- Implementing metal-only changes
- Using spare cells for bug fixes
- Minimizing verification re-spin

---

## Quick Reference

### ECO Types

| Type | Stage | Impact | Typical Use |
|------|-------|--------|-------------|
| Functional | RTL→GDS | Full re-verify | Bug fixes |
| Timing | Netlist | Timing re-analysis | Setup/hold fixes |
| Metal-only | Post-route | Minimal | Late bug fixes |
| Spare cell | Post-route | Minimal | Field fixes |

### ECO Decision Matrix

| Change Type | Preferred Method | Verification Impact |
|-------------|------------------|---------------------|
| Logic bug (early) | RTL change | Full regression |
| Logic bug (late) | Spare cells | Targeted tests |
| Timing fix | Gate sizing/buffering | STA + incremental physical checks |
| Hold fix | Buffer insertion | STA + hold sign-off re-check |
| DFT fix | Spare cells | DFT re-run |

---

## Methodology

### Step 1: ECO Analysis

**Document the issue:**
```
ECO Request: [ID]
├── Description: [brief description]
├── Root cause: [analysis]
├── Affected signals: [list]
├── Affected modules: [list]
├── Design stage: [RTL/Netlist/Layout]
├── Urgency: [Critical/High/Medium/Low]
└── Risk assessment: [impact analysis]
```

**Determine ECO type:**
```
                    ┌─────────────────┐
                    │ What stage is   │
                    │ the design in?  │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
      Pre-syn            Post-syn            Post-layout
         │                   │                   │
         ▼                   ▼                   ▼
    RTL change         Netlist ECO         Metal-only ECO
    (preferred)        or spare cells      or spare cells
```

### Step 2: Impact Assessment

**Affected scope analysis:**
```systemverilog
// Identify all affected logic cones
// Forward trace: What does the changed signal drive?
// Backward trace: What drives the changed signal?

// Example: Changing signal 'ctrl_enable'
// Forward: ctrl_enable → state_machine → output_valid → ...
// Backward: config_reg[0] → ctrl_enable
```

**Verification impact:**
```
Change scope        → Required verification
─────────────────────────────────────────────
Single register     → Targeted unit test
Control path        → FSM coverage re-run
Data path           → Data integrity tests
Clock/reset         → Full regression + CDC
Multiple modules    → Integration + system test
```

### Step 3: ECO Implementation

#### RTL ECO (Pre-synthesis)

**Minimal change principle:**
```systemverilog
// BAD: Restructuring code
// GOOD: Surgical fix

// Original (buggy)
assign data_valid = state == ACTIVE;

// ECO fix (minimal change)
assign data_valid = (state == ACTIVE) && enable;  // Added enable gate
```

**ECO tracking in RTL:**
```systemverilog
// ECO marker for traceability
// ECO_ID: ECO-2024-0042
// ECO_DESC: Fix data_valid assertion during reset
// ECO_DATE: 2024-01-15
// ECO_OWNER: engineer@company.com

// Original: assign data_valid = state == ACTIVE;
assign data_valid = (state == ACTIVE) && !reset_pending;  // ECO-2024-0042
```

#### Netlist ECO

**Common netlist modifications:**
```
1. Gate replacement (resize)
   - Change cell drive strength
   - BUFX2 → BUFX4

2. Buffer insertion
   - Add buffer for timing
   - Add inverter pair for delay

3. Gate insertion
   - Add AND/OR gate for logic fix
   - Use spare cells

4. Net reconnection
   - Change signal source
   - Add/remove connections
```

**Netlist ECO script example:**
```tcl
# ECO: Add enable gating to data_valid
# Find the original driver
set orig_driver [get_pins data_valid_reg/Q]

# Disconnect from load
disconnect_net data_valid_net

# Insert AND gate from spare
set eco_and [get_cells spare_and2_0]
connect_pin $eco_and/A $orig_driver
connect_pin $eco_and/B enable_signal
connect_net data_valid_net [$eco_and/Y]
```

#### Metal-Only ECO

**Constraints:**
- No new base layers (diffusion, poly, well)
- Only metal and via changes
- Must use existing cells (including spares)

**Planning:**
```
Available resources for metal ECO:
├── Spare cells (pre-placed)
│   ├── Spare inverters: 50
│   ├── Spare NANDs: 30
│   ├── Spare NORs: 20
│   └── Spare flops: 10
├── Unused tie cells
└── Feedthrough cells
```

### Step 4: Spare Cell Strategy

**Spare cell planning:**
```systemverilog
module spare_cells (
  input  logic [49:0] spare_inv_in,
  output logic [49:0] spare_inv_out,
  input  logic [29:0] spare_nand_a,
  input  logic [29:0] spare_nand_b,
  output logic [29:0] spare_nand_out,
  input  logic [19:0] spare_nor_a,
  input  logic [19:0] spare_nor_b,
  output logic [19:0] spare_nor_out,
  input  logic        clk,
  input  logic        rst_n,
  input  logic [9:0]  spare_ff_d,
  output logic [9:0]  spare_ff_q
);

  // Spare inverters - tied off by default
  genvar i;
  generate
    for (i = 0; i < 50; i++) begin : gen_spare_inv
      // synthesis attribute keep of spare_inv is true
      INV_X1 spare_inv (.A(spare_inv_in[i]), .Y(spare_inv_out[i]));
    end
  endgenerate

  // Spare NAND gates
  generate
    for (i = 0; i < 30; i++) begin : gen_spare_nand
      NAND2_X1 spare_nand (.A(spare_nand_a[i]), .B(spare_nand_b[i]),
                           .Y(spare_nand_out[i]));
    end
  endgenerate

  // Spare flip-flops
  generate
    for (i = 0; i < 10; i++) begin : gen_spare_ff
      DFFR_X1 spare_ff (.CK(clk), .RN(rst_n), .D(spare_ff_d[i]),
                        .Q(spare_ff_q[i]));
    end
  endgenerate

endmodule
```

**Spare cell placement:**
```
Distribute spare cells throughout the design:

┌────────────────────────────────────┐
│  ░░░    ░░░    ░░░    ░░░    ░░░   │
│       Block A              Block B │
│  ░░░    ░░░    ░░░    ░░░    ░░░   │
│                                    │
│  ░░░    ░░░    ░░░    ░░░    ░░░   │
│       Block C              Block D │
│  ░░░    ░░░    ░░░    ░░░    ░░░   │
└────────────────────────────────────┘
  ░░░ = Spare cell cluster

Rule of thumb: 1-2% of cell area as spares
Cluster size: 5-10 cells per cluster
Distribution: Even spread, near critical blocks
```

**Using spare cells for ECO:**
```tcl
# Find nearest available spare
set target_loc [get_location problem_signal]
set nearest_spare [find_nearest_spare NAND2 $target_loc]

# Connect spare for ECO
connect_spare $nearest_spare \
  -input_a original_signal \
  -input_b eco_enable \
  -output eco_fixed_signal
```

### Step 5: ECO Verification

**Verification strategy by ECO type:**

| ECO Type | Verification Required |
|----------|----------------------|
| RTL functional | Full regression, coverage |
| Netlist timing | STA, timing signoff |
| Metal-only | LVS, DRC, targeted sim |
| Spare cell | Formal equiv + targeted sim |

**Formal equivalence checking:**
```
Pre-ECO netlist vs Post-ECO netlist

Setup:
1. Define ECO boundary
2. Mark intentional differences
3. Run equivalence check
4. Debug any failures
```

**Targeted simulation:**
```systemverilog
// ECO-specific test
class eco_test extends base_test;
  task run_phase(uvm_phase phase);
    // Focus on affected functionality
    eco_scenario seq;
    seq = eco_scenario::type_id::create("seq");

    // Test the specific fix
    seq.test_mode = ECO_CONDITION;
    seq.start(env.agent.sequencer);

    // Test boundary conditions
    seq.test_mode = ECO_BOUNDARY;
    seq.start(env.agent.sequencer);
  endtask
endclass
```

---

## ECO Documentation

**Required documentation:**
```
ECO Package: ECO-YYYY-NNNN
├── ECO Request Form
│   ├── Problem description
│   ├── Root cause analysis
│   └── Risk assessment
├── Technical Specification
│   ├── Affected signals/modules
│   ├── Logic changes (schematic)
│   └── Implementation method
├── Verification Plan
│   ├── Tests to run
│   ├── Coverage requirements
│   └── Sign-off criteria
├── Implementation Log
│   ├── Files changed
│   ├── Commands executed
│   └── Before/after comparison
└── Sign-off Record
    ├── Design review
    ├── Verification sign-off
    └── Quality approval
```

---

## Checklist

### ECO Request
- [ ] Problem clearly described
- [ ] Root cause identified
- [ ] Affected scope documented
- [ ] Risk assessment completed
- [ ] Urgency justified

### ECO Implementation
- [ ] Minimal change principle followed
- [ ] ECO markers added to code
- [ ] Change reviewed by second engineer
- [ ] Implementation matches specification

### ECO Verification
- [ ] Verification plan approved
- [ ] Required tests executed
- [ ] Coverage requirements met
- [ ] Formal equivalence passed (if applicable)
- [ ] Timing clean (STA)

### ECO Closure
- [ ] All documentation complete
- [ ] Files checked into version control
- [ ] ECO tracking system updated
- [ ] Lessons learned captured

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|-------------|------------|
| Incomplete impact analysis | Missed side effects | Trace all affected paths |
| Insufficient spares | Cannot implement fix | Plan spares at RTL stage |
| Poor spare placement | Routing congestion | Distribute evenly |
| Skipping formal check | Functional regression | Always run equiv check |
| Inadequate documentation | Knowledge loss | Document as you go |

---

## Resources

- **References:**
  - `references/eco-methodology.md` - ECO classification, decision flow, verification matrix, documentation template
  - `references/spare-cell-planning.md` - Cell type mix, budget rules, placement strategy, naming convention

- **Examples:**
  - `examples/example-functional-eco.md` - Netlist ECO using spare NAND2 with full verification checklist
