---
name: rtl-code-review
description: >-
  This skill handles requests to "review RTL code", "review this Verilog",
  "review this SystemVerilog", "check FSM for bugs", "find CDC violations", "analyze this
  hardware module", "check for silicon failure bugs", "review this ASIC design",
  "check this module for bugs", "check AXI protocol compliance", "lint RTL",
  "check this RTL", or any RTL/hardware
  design code review. Provides systematic IC designer methodology for finding silicon
  failure bugs including FSM deadlock, data corruption, CDC violations, and protocol errors.
---

# RTL Code Review

## Quick Mode
When user asks for a quick review, focus on these 3 checks only:
1. **FSM reachability** — Can every state be reached and exited? Any deadlock?
2. **Reset check** — Are all stateful registers properly reset?
3. **Protocol compliance** — AXI/handshake VALID/READY rules violated?

---

Systematic RTL code review methodology for finding silicon failure bugs in Verilog/SystemVerilog designs.

## Core Principle

**Focus on silicon failure bugs, not coding style.** Prioritize FSM deadlock, data corruption, CDC violations, and protocol errors. Style issues are informational only.

---

## IC Designer Methodology (6-Step Process)

Follow these steps IN ORDER for every review. Do not skip to pattern matching.

### Step 1: Understand Design Intent

Before reading any code, establish:
- Module function and role in the system (controller, datapath, interface, memory)
- Clock domains involved
- Key control signals and their **complete set of valid values**

### Step 2: Exhaustive Path Tracing

For **each valid value** of each control signal, trace the complete execution path through the FSM:

```
control_signal = value_0 → FSM path? Actions triggered? Final output?
control_signal = value_1 → FSM path? Actions triggered? Final output?
...
```

Enumerate ALL combinations. Missing a single encoding is the #1 cause of silicon bugs.

### Step 3: Manual Boolean Calculation

Substitute concrete values into every conditional expression. Verify no condition is always TRUE or always FALSE for a valid input:

```verilog
// exit_cond = (shared_tree | (chroma_tree_d && !chroma_tree))
// When cu_tree_type=1: shared_tree=0, chroma_tree=0
// exit_cond = (0 | (0 && 1)) = 0 → Always FALSE → DEADLOCK
```

**If a valid input makes a condition constant, this is almost always a bug.**

**Operator precedence awareness** — Verilog precedence (high→low): `&` > `^` > `|` > `&&` > `||`. When analyzing expressions with mixed operators:
1. **Respect explicit parentheses first** — explicit `()` always override implicit precedence. Parse the expression outward from innermost parentheses. Never re-group tokens that are already inside explicit parentheses.
2. **Don't assume it's a bug** — the implicit precedence may match the designer's intent
3. **Only flag as bug** if you have concrete evidence (e.g., from simulation behavior, signal naming, or surrounding logic) that the parsing doesn't match the intended behavior
4. **If uncertain, report as Observation** — suggest adding parentheses for readability, not as a bug fix
5. **Never change operator types** in suggested fixes (`&` → `&&` or `|` → `||`) — they have different semantics on multi-bit signals. Always suggest adding parentheses only.

### Step 4: Action-Transition Consistency

For each FSM state transition:
- Compare the Boolean condition controlling the transition against the condition controlling the associated data-path action
- If the transition fires but the action does not (or vice versa), flag as inconsistency

### Step 5: Evaluate Silicon Consequences

Classify each finding by consequence:

| Consequence | Severity | Examples |
|-------------|----------|----------|
| Silicon failure (unrecoverable) | **Critical** | CDC violation, multi-driver, spec non-compliance |
| Functional error (wrong output) | **High** | Missing encoding, RAW hazard, protocol deadlock |
| Edge case issue | **Medium** | Unguarded overflow, missing timeout |
| Style / maintainability | **Low** | Naming, comments, code organization |

Report only Critical and High issues with full confidence. Mark uncertain findings as Observations.

### Step 6: Cross-Check Against Known Patterns

After completing Steps 1-5, consult `references/bug-patterns.md` for known bug signatures organized by category (CDC, FSM, Memory, Bus Protocol, Video Codec). Cross-check findings against these patterns and scan for any missed issues.

---

## Review Output Format

Structure every review report as follows:

```markdown
## [Module Name] RTL Review

### Design Understanding
- Function: [what the module does]
- Key control signals: [signal name] — valid values: [enumerate all]
- Clock domains: [list]

### Critical Issues (Silicon Failure)
1. [Line#] **Issue title**
   - Trigger condition: when [signal] = [value]
   - Root cause: [why it fails — condition always TRUE/FALSE, missing encoding, etc.]
   - Consequence: [FSM deadlock / data corruption / protocol violation / ...]
   - Suggested fix: [specific code change]

### High Issues (Functional Error)
[same format]

### Medium/Low Issues
[brief description, no full analysis needed]

### Observations
- Design notes and questions for the author
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Missing input combinations | Unhandled case values create latches | Check all possible input values; verify `default` branch exists |
| Style over function focus | Reviewer catches naming issues but misses deadlocks | Prioritize silicon bugs (deadlock, overflow, race) over style |
| Missing inter-module dependencies | Data may not be stable at capture time | Verify timing across module boundaries; check valid-data alignment |
| Incomplete reset analysis | Some registers start at X in gate simulation | Check every stateful register has proper reset value |
| Reviewing modules in isolation | Cross-module protocol violations escape | Review interfaces between modules together |
| Ignoring synthesis implications | Code works in simulation but fails in silicon | Check for latches, combinational loops, multi-driven nets |
| Not checking clock domains | CDC bugs are the #1 silicon failure | Flag any signal crossing clock boundaries without synchronizer |
| Ignoring explicit parentheses | Agent re-groups tokens across `()` boundaries when analyzing precedence | Always parse from innermost explicit `()` outward. Never move operators inside or outside existing parentheses during analysis. |
| Flagging precedence as bug | Mixed `&`/`&&`/`|`/`||` without parens may be intentional | Only flag as bug with concrete evidence of incorrect behavior; otherwise report as Observation suggesting parentheses for readability. Never change operator types in fixes. |

---

## Review Report Template

```systemverilog
//=============================================================================
// Review Checklist Template - Copy and fill for each module
//=============================================================================
// Module: _______________
// Reviewer: _____________
// Date: _________________
//
// [ ] 1. FSM Analysis
//     - All states reachable?
//     - All states have exit condition?
//     - Default/illegal state handling?
//
// [ ] 2. Reset Analysis  
//     - All registers reset?
//     - Reset value correct?
//     - Async/sync reset consistent?
//
// [ ] 3. Clock Domain Analysis
//     - Single clock or multi-clock?
//     - CDC signals identified?
//     - Synchronizers present?
//
// [ ] 4. Protocol Compliance
//     - VALID before READY dependency?
//     - Handshake rules followed?
//     - Burst/transaction boundaries?
//
// [ ] 5. Data Integrity
//     - Width mismatches?
//     - Sign extension correct?
//     - Overflow handling?
//
// [ ] 6. Latch Check
//     - All if/case complete?
//     - Combinational outputs driven in all paths?
//=============================================================================
```

---

## Verification Checklist

- [ ] All valid input combinations traced through FSM
- [ ] Boolean conditions manually evaluated for edge cases
- [ ] Reset initialization verified for all stateful elements
- [ ] CDC crossings identified and synchronization verified
- [ ] Protocol compliance checked (AXI/AHB/handshake rules)
- [ ] No potential deadlock conditions
- [ ] Width mismatches identified
- [ ] Overflow/underflow conditions checked
- [ ] Findings classified by severity (Critical/High/Medium/Low)
- [ ] False positives filtered with concrete trigger conditions

---

## False Positive Avoidance

Before reporting any issue, verify:

1. **Concrete trigger**: Identify a specific input value and state that causes the failure. No concrete trigger = not a real bug.
2. **Reachability**: Confirm the trigger condition is actually reachable in normal operation.
3. **Design intent**: Distinguish intentional design choices from actual errors. When uncertain, report as Observation.
4. **Sufficient context**: If the module is part of a larger system and behavior depends on external constraints, note the assumption rather than flagging as bug.
5. **Operator precedence**: Mixed bitwise/logical operators without parentheses are NOT automatically bugs. The implicit Verilog precedence (`&` > `|` > `&&` > `||`) may be exactly what the designer intended. Only flag as a bug when you can demonstrate the parsing produces incorrect behavior. Otherwise, report as a readability Observation.

---

## Additional Resources

### Reference Files

For detailed bug patterns with code examples, consult:
- **`references/bug-patterns.md`** — Complete bug pattern catalog organized by category: CDC, FSM, Memory, Bus Protocol, Video Codec. Each pattern includes detection method, severity, and Verilog code examples.
- **`references/axi-protocol-checklist.md`** — AXI4 protocol compliance checklist: handshake rules, burst boundaries, channel ordering, deadlock patterns, RESP encoding, and reset requirements.

### Example Files

For expected review output quality and format:
- **`examples/example-review.md`** — Complete review of a frame buffer controller FSM, demonstrating the 6-step methodology applied to find 3 real bugs (write phase missing, deadlock on write-only mode, idle mode acceptance).
