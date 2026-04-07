---
name: debug-methodology
description: >-
  This skill handles RTL debugging tasks including simulation failures,
  waveform analysis, protocol violations, and silicon bring-up issues.
  Use for "debug failure", "find bug", "analyze waveform", "root cause",
  or any debugging and troubleshooting task.
---

# Debug Methodology Skill

Systematic approaches for debugging RTL designs, simulations, and silicon issues.

## Quick Mode
When user asks for quick debug help, focus on these 2 checks only:
1. **Reproduce the failure** — Get a deterministic test case that triggers the bug every run
2. **Find first cycle of failure** — Binary-search waveform to pinpoint the exact cycle things go wrong

---

## When to Use

- Simulation mismatches or failures
- Unexpected waveform behavior
- Silicon bring-up issues
- Performance anomalies
- Protocol violations
- Intermittent failures

---

## Methodology

### Step 1: Reproduce and Isolate

**Reproduce reliably:**
1. Capture exact test conditions (seed, configuration)
2. Minimize test case to smallest failing scenario
3. Determine if failure is deterministic or intermittent

**Isolate the scope:**
```
System Level → Subsystem → Block → Signal
     ↓              ↓          ↓        ↓
  Full SoC      Memory     Cache    tag_valid
              Subsystem   Controller
```

### Step 2: Hypothesis Formation

**Common root causes:**
| Category | Examples |
|----------|----------|
| Reset | Missing reset, wrong polarity, async reset release |
| Clock | Gating issues, CDC bugs, frequency mismatch |
| Protocol | Handshake violations, ordering errors |
| Data | Wrong width, sign extension, endianness |
| Timing | Setup/hold violations, race conditions |
| Logic | Off-by-one, boundary conditions, state machine |

### Step 3: Bisection Debug

**Binary search through:**
- Time: When does the bug first appear?
- Space: Which module causes the issue?
- Configuration: Which setting triggers the bug?

```
Time Bisection:
|-----------------------------------------------|
0                    T_fail                   T_end
         |
    First good → search here → first bad
```

### Step 4: Root Cause Analysis

**5 Whys technique:**
```
Problem: Data corruption
Why 1: FIFO overflow
Why 2: Backpressure not working
Why 3: Ready signal stuck low
Why 4: FSM stuck in WAIT state
Why 5: Missing transition from WAIT to IDLE (ROOT CAUSE)
```

### Step 5: Fix and Verify

1. Implement minimal fix
2. Verify fix resolves original failure
3. Run regression to check for side effects
4. Add assertion to catch future occurrence
5. Document the bug and fix

---

## Debug Techniques

### Waveform Analysis

**Key signals to trace:**
```
1. Clock and reset
2. FSM state
3. Valid/ready handshakes
4. Error indicators
5. Data paths of interest
```

**Markers to set:**
- First error occurrence
- Last known good state
- State transitions of interest

### Printf/Display Debug

```systemverilog
`ifdef DEBUG
always @(posedge clk) begin
    if (state != state_prev)
        $display("[%0t] %m: state %s -> %s", $time, state_prev.name(), state.name());
    if (error)
        $display("[%0t] %m: ERROR - %s", $time, error_msg);
end
`endif
```

### Assertion-Based Debug

```systemverilog
// Catch the bug in action
assert property (@(posedge clk) disable iff (!rst_n)
    fifo_count <= FIFO_DEPTH
) else $error("FIFO overflow at %0t", $time);

// Track signal history
property no_back_to_back_errors;
    @(posedge clk) error |=> !error;
endproperty
assert property (no_back_to_back_errors);
```

### Force/Release

```systemverilog
// Temporarily force signals to test theories
initial begin
    #1000;
    force dut.fsm.state = IDLE;  // Force FSM reset
    #10;
    release dut.fsm.state;
end
```

---

## Quick Reference

### Debug Checklist

- [ ] Can reproduce the failure?
- [ ] Failure deterministic or random?
- [ ] First cycle of failure identified?
- [ ] Scope narrowed to specific module?
- [ ] Hypothesis formed and tested?
- [ ] Root cause identified?
- [ ] Fix verified?
- [ ] Regression passed?
- [ ] Assertion added to prevent recurrence?

### Debug Commands (Simulation)

```tcl
# VCS
run -all
force {top.signal} 1
release {top.signal}
describe {top.module.*}

# Questa
run -all
force /top/signal 1
noforce /top/signal
examine -radix hex /top/data
```

---

## Checklist

- [ ] Failure reproduced
- [ ] Test case minimized
- [ ] Time of first failure found
- [ ] Faulty module identified
- [ ] Root cause determined
- [ ] Fix implemented
- [ ] Regression clean
- [ ] Assertion added

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| X propagation | Uninitialized register drives unknown | Add proper reset or initial values |
| Glitches | Combinational hazard on outputs | Register outputs or add glitch filters |
| Data 1 cycle late | Pipeline stage mismatch | Trace pipeline depth, align valid/data |
| Works in isolation | Integration timing/protocol issue | Test with full system context |
| Fails only with optimization | Timing race or tool artifact | Run with `-debug_access` or disable opt |
| Random failures | CDC or metastability | Add synchronizers, check CDC paths |
| Waveform too short | Bug happens late in simulation | Increase dump window, use trigger-based dump |
| Wrong signal hierarchy | Force/examine wrong instance | Use `find` to verify signal path first |

---

## Resources

- **References:**
  - `references/waveform-debug.md` - Waveform analysis techniques

- **Examples:**
  - `examples/example-fifo-debug.md` - FIFO overflow debug with 5-Whys root cause and assertion fix
