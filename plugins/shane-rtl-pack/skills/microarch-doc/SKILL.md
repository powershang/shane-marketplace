---
name: microarch-doc
description: >-
  This skill handles requests to "write microarchitecture document",
  "create design spec", "document RTL", "write architecture description",
  "create block diagram", "document interface", "write register map",
  "create timing diagram", "document FSM", "write design document",
  or any microarchitecture documentation task. Provides systematic methodology
  for creating clear, complete, and maintainable hardware design documentation.
---

# Microarchitecture Documentation

## Quick Mode
When user asks for quick doc help, focus on these 2 items only:
1. **Block diagram** — High-level view with major components and data flow
2. **Interface table** — All ports with direction, width, and description

---

Systematic microarchitecture documentation methodology for clear, maintainable design specs.

## Core Principle

**Documentation is the contract between design and verification.** A good microarch doc enables others to understand, verify, and modify your design without reading RTL. It should answer: what does this do, how does it do it, and why was it designed this way?

---

## Documentation Methodology (5-Section Structure)

Every microarchitecture document should contain these sections:

### Section 1: Overview

**Purpose:** One paragraph explaining what the module does and its role in the system.

**Template:**
```markdown
## 1. Overview

The `<module_name>` implements <primary function>. It interfaces with
<upstream module> to receive <input data type> and provides <output>
to <downstream module>. Key features include:
- <Feature 1>
- <Feature 2>
- <Feature 3>

### 1.1 Block Diagram

[Insert block diagram showing this module in system context]

### 1.2 Key Specifications

| Parameter | Value |
|-----------|-------|
| Clock frequency | XXX MHz |
| Throughput | XXX per cycle |
| Latency | XXX cycles |
| Area estimate | XXX gates |
```

### Section 2: Interface

**Purpose:** Complete port list with detailed descriptions.

**Template:**
```markdown
## 2. Interface

### 2.1 Port List

| Port Name | Direction | Width | Description |
|-----------|-----------|-------|-------------|
| clk | input | 1 | System clock |
| rst_n | input | 1 | Active-low async reset |
| ... | ... | ... | ... |

### 2.2 Interface Timing

#### Write Interface
- `wr_valid` asserted for one cycle per write
- `wr_data` must be stable when `wr_valid` is high
- `wr_ready` indicates module can accept data

[Include timing diagram if protocol is non-trivial]

### 2.3 Interface Protocol

[Describe handshake, backpressure, or special sequencing requirements]
```

### Section 3: Microarchitecture

**Purpose:** Internal structure and data flow.

**Template:**
```markdown
## 3. Microarchitecture

### 3.1 Block Diagram

```
┌─────────────────────────────────────────────────┐
│                  Module Name                     │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│  │ Stage 1 │───▶│ Stage 2 │───▶│ Stage 3 │     │
│  └─────────┘    └─────────┘    └─────────┘     │
│       │              │              │           │
│       ▼              ▼              ▼           │
│  ┌─────────────────────────────────────────┐   │
│  │            Control FSM                   │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 3.2 Data Path

[Describe the main data transformation pipeline]

### 3.3 Control Path

[Describe FSM, control signals, sequencing]

### 3.4 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Registered outputs | Better timing closure |
| 2-deep input buffer | Handle backpressure |
| Gray-code pointers | Safe CDC crossing |
```

### Section 4: Functional Description

**Purpose:** Detailed behavior for each feature/mode.

**Template:**
```markdown
## 4. Functional Description

### 4.1 Normal Operation

[Step-by-step description of typical transaction flow]

1. Master asserts `req` with valid `addr` and `data`
2. Module registers request on next clock edge
3. [...]

### 4.2 Special Modes

#### 4.2.1 Bypass Mode
When `bypass_en` is high:
- Data flows directly from input to output
- Latency reduced to 1 cycle
- [...]

### 4.3 Error Handling

| Error Condition | Detection | Response |
|-----------------|-----------|----------|
| FIFO overflow | `wr_en && full` | Assert `error`, drop data |
| Invalid address | `addr > MAX_ADDR` | Return error response |

### 4.4 Reset Behavior

On assertion of `rst_n`:
- All state registers → 0
- FSM → IDLE state
- Output ports → deasserted
- FIFO pointers → 0
```

### Section 5: Implementation Details

**Purpose:** Information needed for verification and integration.

**Template:**
```markdown
## 5. Implementation Details

### 5.1 Clock Domains

| Domain | Signals | Crossing Method |
|--------|---------|-----------------|
| clk_core | All datapath | - |
| clk_cfg | Config registers | 2FF sync for control |

### 5.2 Register Map

| Offset | Name | Access | Reset | Description |
|--------|------|--------|-------|-------------|
| 0x00 | CTRL | RW | 0x0 | Control register |
| 0x04 | STATUS | RO | 0x0 | Status register |
| 0x08 | DATA | RW | 0x0 | Data register |

#### CTRL Register (0x00)

| Bits | Field | Access | Reset | Description |
|------|-------|--------|-------|-------------|
| [0] | EN | RW | 0 | Module enable |
| [1] | MODE | RW | 0 | Operating mode |
| [7:2] | RSVD | RO | 0 | Reserved |

### 5.3 FSM States

| State | Encoding | Description | Exit Conditions |
|-------|----------|-------------|-----------------|
| IDLE | 3'b001 | Wait for request | `req` → ACTIVE |
| ACTIVE | 3'b010 | Processing | `done` → IDLE |
| ERROR | 3'b100 | Error state | `clr_err` → IDLE |

### 5.4 Timing Constraints

- Input setup: Data must be stable 0.5ns before clock edge
- Output delay: Data valid 1.2ns after clock edge
- Multicycle paths: `cfg_*` registers are 2-cycle paths
```

---

## Diagram Guidelines

### Block Diagrams (ASCII)

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Input   │────▶│ Process  │────▶│  Output  │
│  Buffer  │     │  Unit    │     │  Buffer  │
└──────────┘     └──────────┘     └──────────┘
      │               │                │
      └───────────────┴────────────────┘
                      │
              ┌───────▼───────┐
              │   Control     │
              └───────────────┘
```

### Timing Diagrams (WaveDrom JSON)

```json
{ "signal": [
  { "name": "clk",     "wave": "p......." },
  { "name": "req",     "wave": "0.1..0.." },
  { "name": "data",    "wave": "x.=..x..", "data": ["D0"] },
  { "name": "ack",     "wave": "0...1.0." },
  { "name": "busy",    "wave": "0.1...0." }
]}
```

### FSM Diagrams (ASCII)

```
        ┌─────┐
   ─────▶ IDLE │◀──────────┐
        └──┬──┘           │
     req=1 │              │ done=1
           ▼              │
        ┌─────┐        ┌──┴──┐
        │ RUN │───────▶│DONE │
        └─────┘        └─────┘
```

---

## Report Output Format

When documenting a module, produce:

```markdown
# <Module Name> Microarchitecture Specification

**Version:** 1.0
**Author:** <name>
**Date:** <date>
**Status:** Draft / Review / Final

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | YYYY-MM-DD | Name | Initial version |

## 1. Overview
[...]

## 2. Interface
[...]

## 3. Microarchitecture
[...]

## 4. Functional Description
[...]

## 5. Implementation Details
[...]

## Appendix A: Verification Considerations
- Key scenarios to test
- Coverage points
- Known limitations
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Doc-RTL mismatch | Documentation becomes stale | Update doc with every RTL change |
| Missing corner cases | Verification gaps | Document all special/error conditions |
| Ambiguous timing | Misinterpretation by verifier | Use precise cycle-by-cycle descriptions |
| No rationale | Future engineers don't understand decisions | Always explain "why", not just "what" |
| Missing reset behavior | Integration issues | Explicitly document all reset values |
| Interface timing unclear | Protocol violations | Include timing diagrams for all interfaces |
| Register side effects | Software bugs | Document all read/write side effects |

---

## Additional Resources

### Reference Files

For documentation templates:
- **`references/doc-template.md`** — Complete microarchitecture document template ready to fill in.

### Example Files

For expected documentation quality:
- **`examples/example-microarch-doc.md`** — Complete microarchitecture spec for a DMA controller, demonstrating all sections.
