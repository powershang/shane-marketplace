---
name: fsm-design
description: >-
  This skill handles requests to "design FSM", "create state machine",
  "write controller", "implement sequencer", "FSM encoding", "state diagram",
  "fix FSM deadlock", "optimize state machine", "one-hot encoding",
  "binary encoding", "Mealy machine", "Moore machine", "FSM verification",
  or any finite state machine design task. Provides systematic methodology
  for designing robust, verifiable, and synthesizable state machines.
---

# FSM Design

## Quick Mode
When user asks for quick FSM help, focus on these 2 checks only:
1. **Deadlock check** — Can every state be exited under some input condition?
2. **Illegal state handling** — What happens if FSM enters undefined state?

---

Systematic FSM design methodology for robust, verifiable, and synthesis-friendly state machines.

## Core Principle

**An FSM is only as reliable as its weakest transition.** Every state must have a defined exit path for every possible input combination. Unhandled cases lead to deadlock or unpredictable behavior that may only surface in silicon.

---

## FSM Design Methodology (5-Step Process)

### Step 1: Requirements Analysis

Before writing any code, define:

| Item | Questions to Answer |
|------|---------------------|
| **Inputs** | What signals trigger transitions? Are they synchronous? |
| **Outputs** | Moore (output = f(state)) or Mealy (output = f(state, input))? |
| **States** | How many distinct states? Can they be decomposed? |
| **Timing** | Max cycles in any state? Timeout requirements? |
| **Error handling** | How to recover from illegal states? |

### Step 2: State Diagram Design

Create a complete state transition diagram:

```
     ┌─────────────────────────────────────────┐
     │                                         │
     ▼                                         │
  ┌──────┐  start=1   ┌──────┐  done=1   ┌──────┐
  │ IDLE │───────────▶│ BUSY │──────────▶│ DONE │
  └──────┘            └──────┘            └──────┘
     ▲                    │                   │
     │                    │ error=1           │ ack=1
     │                    ▼                   │
     │               ┌───────┐                │
     └───────────────│ ERROR │◀───────────────┘
                     └───────┘
                       timeout
```

**State Diagram Checklist:**

| Check | Pass Criteria |
|-------|---------------|
| All states reachable | Every state has at least one incoming transition |
| All states exitable | Every state has at least one outgoing transition |
| All inputs considered | Every state handles every possible input combination |
| Reset defined | Clear reset state identified |
| Default transitions | Unspecified conditions have explicit handling |

### Step 3: Encoding Selection

Choose encoding based on design goals:

| Encoding | States | FFs | Speed | Area | Power | Best For |
|----------|--------|-----|-------|------|-------|----------|
| **One-hot** | N | N | Fast | Large | Medium | FPGA, speed-critical |
| **Binary** | N | ⌈log₂N⌉ | Slow | Small | Low | ASIC, area-critical |
| **Gray** | N | ⌈log₂N⌉ | Medium | Small | Low | CDC, low-power |
| **Custom** | N | varies | varies | varies | varies | Special requirements |

**Encoding Decision Tree:**

```
Is FPGA target?
├── Yes → Use one-hot (free FFs, fast decode)
└── No (ASIC)
    ├── Is FSM in critical timing path?
    │   ├── Yes → Use one-hot
    │   └── No → Use binary
    └── Does FSM cross clock domain?
        └── Yes → Consider Gray encoding for state bits
```

### Step 4: Implementation

Select implementation style based on complexity:

| Style | Complexity | Readability | Synthesis QoR |
|-------|------------|-------------|---------------|
| **1-process** | Simple FSM | Lower | Good |
| **2-process** | Medium FSM | Higher | Good |
| **3-process** | Complex FSM | Highest | Best |

**Recommended: 2-process style** for most designs:

```systemverilog
// Process 1: State register (sequential)
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        state <= IDLE;
    else
        state <= next_state;
end

// Process 2: Next state + outputs (combinational)
always_comb begin
    // Defaults (prevent latches)
    next_state = state;
    output_a = 1'b0;
    output_b = 1'b0;
    
    case (state)
        IDLE: begin
            if (start)
                next_state = BUSY;
        end
        BUSY: begin
            output_a = 1'b1;
            if (done)
                next_state = DONE;
            else if (error)
                next_state = ERROR;
        end
        // ... more states
        default: next_state = IDLE;  // Illegal state recovery
    endcase
end
```

### Step 5: Verification

Add assertions to verify FSM correctness:

```systemverilog
// 1. No illegal states
assert property (@(posedge clk) disable iff (!rst_n)
    state inside {IDLE, BUSY, DONE, ERROR}
) else $error("Illegal state: %0d", state);

// 2. No deadlock - every state eventually exits (liveness)
assert property (@(posedge clk) disable iff (!rst_n)
    state == BUSY |-> ##[1:MAX_CYCLES] state != BUSY
) else $error("Deadlock in BUSY state");

// 3. Mutual exclusion for one-hot
assert property (@(posedge clk) disable iff (!rst_n)
    $onehot(state)
) else $error("One-hot violation");

// 4. Valid transitions only
assert property (@(posedge clk) disable iff (!rst_n)
    (state == IDLE) && (next_state != IDLE) |-> (next_state == BUSY)
) else $error("Invalid transition from IDLE");
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Incomplete case statement | Missing states/default infers latches | Default assign `next_state = state` before case; add `default` branch |
| Missing exit condition | State deadlocks if expected signal never asserts | Add timeout counter as escape path to ERROR state |
| Glitchy Mealy outputs | Combinational outputs glitch during input transitions | Register all outputs for clean timing |
| Incomplete reset | Some stateful signals start at X | Reset ALL registers (state, counters, flags) in reset block |
| Unreachable states | Dead states waste encoding bits | Verify all states reachable via formal or simulation coverage |
| No illegal state recovery | Bit-flip puts FSM in undefined state | `default` branch returns to IDLE/safe state |
| One-hot encoding without full_case | Synthesis assumes mutual exclusivity incorrectly | Use `unique case` or add explicit priority |

---

## FSM Design Patterns

### Pattern 1: Pipeline Controller

```systemverilog
typedef enum logic [2:0] {
    PIPE_IDLE,
    PIPE_STAGE1,
    PIPE_STAGE2,
    PIPE_STAGE3,
    PIPE_DONE
} pipe_state_t;

// Each stage advances on valid handshake
always_comb begin
    next_state = state;
    case (state)
        PIPE_IDLE:   if (valid_in)  next_state = PIPE_STAGE1;
        PIPE_STAGE1: if (stage1_done) next_state = PIPE_STAGE2;
        PIPE_STAGE2: if (stage2_done) next_state = PIPE_STAGE3;
        PIPE_STAGE3: if (stage3_done) next_state = PIPE_DONE;
        PIPE_DONE:   if (ready_out) next_state = PIPE_IDLE;
        default: next_state = PIPE_IDLE;
    endcase
end
```

### Pattern 2: Hierarchical FSM (Complex Protocol)

```systemverilog
// Main FSM
typedef enum logic [1:0] {MAIN_IDLE, MAIN_READ, MAIN_WRITE} main_state_t;

// Sub-FSM for read operation
typedef enum logic [1:0] {RD_ADDR, RD_DATA, RD_RESP} read_state_t;

// Sub-FSM for write operation  
typedef enum logic [1:0] {WR_ADDR, WR_DATA, WR_RESP} write_state_t;

// Main FSM delegates to sub-FSMs
always_comb begin
    case (main_state)
        MAIN_READ:  sub_fsm_active = 1'b1;  // read_state controls
        MAIN_WRITE: sub_fsm_active = 1'b1;  // write_state controls
        default:    sub_fsm_active = 1'b0;
    endcase
end
```

### Pattern 3: Timeout Watchdog

```systemverilog
// Timeout counter runs in parallel
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        timeout_cnt <= '0;
    else if (state == IDLE)
        timeout_cnt <= '0;  // Reset on idle
    else if (state == WAIT)
        timeout_cnt <= timeout_cnt + 1;  // Count during wait
end

wire timeout = (timeout_cnt == TIMEOUT_MAX);

// FSM uses timeout as escape
always_comb begin
    case (state)
        WAIT: begin
            if (ack)
                next_state = DONE;
            else if (timeout)
                next_state = ERROR;  // Escape deadlock
        end
        // ...
    endcase
end
```

---

## Report Output Format

When designing an FSM, produce:

```markdown
# FSM Design: <Module Name>

## Requirements
- Function: [what the FSM controls]
- Inputs: [list with descriptions]
- Outputs: [list with Moore/Mealy classification]
- Timing: [cycle requirements]

## State Diagram
[ASCII or description]

## State Table

| State | Input Condition | Next State | Outputs |
|-------|-----------------|------------|---------|
| IDLE | start=1 | BUSY | ready=0 |
| ... | ... | ... | ... |

## Encoding
- Chosen: [One-hot/Binary/Gray]
- Rationale: [why]

## Implementation
[Verilog code]

## Verification
- Assertions included: [list]
- Coverage points: [state transitions to cover]
```

---

## Additional Resources

### Reference Files

For detailed patterns and comparisons:
- **`references/fsm-encoding.md`** — Encoding strategies with area/timing tradeoffs
- **`references/fsm-styles.md`** — 1/2/3-process implementation styles with code
- **`references/fsm-verification.md`** — Assertion patterns for FSM verification
- **`references/fsm-optimization.md`** — State minimization and optimization techniques

### Example Files

For expected implementation quality:
- **`examples/example-protocol-fsm.md`** — Complete AXI4-Lite slave FSM design
