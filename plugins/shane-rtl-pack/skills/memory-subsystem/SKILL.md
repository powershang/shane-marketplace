---
name: memory-subsystem
description: >-
  This skill handles memory subsystem design including caches, FIFOs, buffers,
  and memory controllers. Use for "design cache", "FIFO implementation", "memory
  controller", "buffer design", "scratchpad", or any memory-related design task.
---

# Memory Subsystem Design Skill

Design and implement memory subsystems including caches, buffers, FIFOs, and memory controllers.

## Quick Mode
When user asks for quick memory help, focus on these 2 checks only:
1. **FIFO sizing** — Verify depth handles worst-case burst + rate mismatch without overflow
2. **Read-write hazard** — Confirm defined behavior for simultaneous read and write to same address

---

## When to Use

- Designing cache hierarchies (L1/L2/L3)
- Implementing FIFOs and circular buffers
- Creating memory controllers (DDR, SRAM, Flash)
- Building scratchpads and tightly-coupled memories
- Designing prefetchers and memory schedulers

---

## Methodology

### Step 1: Requirements Analysis

**Key questions:**
1. What is the memory type? (SRAM, register file, external DRAM)
2. What are the access patterns? (sequential, random, burst)
3. What is the bandwidth requirement? (GB/s)
4. What is the latency budget? (cycles)
5. What is the capacity? (KB, MB)
6. Single-port or multi-port?
7. Clock domain crossings?

**Document in spec:**
```
Memory Subsystem: [name]
├── Type: [SRAM/RegFile/DRAM controller]
├── Capacity: [size]
├── Data width: [bits]
├── Address width: [bits]
├── Ports: [read/write configuration]
├── Latency: [cycles]
├── Bandwidth: [GB/s @ frequency]
└── Special features: [ECC, parity, banking]
```

### Step 2: Architecture Selection

Choose appropriate architecture based on requirements:

| Requirement | Architecture |
|-------------|--------------|
| Low latency, small | Register file |
| Medium size, single access | Single-port SRAM |
| High bandwidth | Multi-bank SRAM |
| Very large capacity | DRAM with controller |
| Variable depth | FIFO |
| Streaming | Circular buffer |
| Temporal locality | Cache |

### Step 3: Microarchitecture Definition

**For caches:**
```
Cache Configuration
├── Organization: [direct-mapped/set-associative/fully-associative]
├── Line size: [bytes]
├── Total size: [KB]
├── Number of sets: [calculated]
├── Ways: [associativity]
├── Replacement: [LRU/PLRU/random/FIFO]
├── Write policy: [write-through/write-back]
├── Allocation: [write-allocate/no-write-allocate]
└── Coherence: [none/MESI/MOESI]
```

**For FIFOs:**
```
FIFO Configuration
├── Depth: [entries]
├── Width: [bits]
├── Type: [sync/async]
├── Almost full threshold: [entries]
├── Almost empty threshold: [entries]
└── Overflow handling: [drop/block/overwrite]
```

**For memory controllers:**
```
Controller Configuration
├── Protocol: [DDR3/DDR4/LPDDR4/HBM]
├── Data width: [bits]
├── Burst length: [beats]
├── Command scheduling: [FCFS/FR-FCFS/PAR-BS]
├── Address mapping: [row-bank-column/bank-row-column]
├── Refresh: [all-bank/per-bank]
└── Power modes: [active/precharge/self-refresh]
```

### Step 4: Implementation

See references for implementation patterns:
- `references/cache-design.md` - Cache architectures
- `references/fifo-design.md` - FIFO implementations
- `references/memory-controller.md` *(planned)* - Controller patterns
- `references/ecc-parity.md` *(planned)* - Error protection

### Step 5: Verification

**Functional coverage:**
- All address ranges accessed
- Boundary conditions (full, empty, wraparound)
- Replacement policy scenarios
- Error injection and recovery

**Performance verification:**
- Bandwidth measurement
- Latency distribution
- Hit/miss rates (for caches)
- Queue depths (for buffers)

---

## Quick Reference

### SRAM Interface
```systemverilog
module sram_1rw #(
    parameter DEPTH = 1024,
    parameter WIDTH = 32
)(
    input  logic                     clk,
    input  logic                     ce,     // Chip enable
    input  logic                     we,     // Write enable
    input  logic [$clog2(DEPTH)-1:0] addr,
    input  logic [WIDTH-1:0]         wdata,
    output logic [WIDTH-1:0]         rdata
);
```

### Synchronous FIFO Interface
```systemverilog
module sync_fifo #(
    parameter DEPTH = 16,
    parameter WIDTH = 32
)(
    input  logic             clk,
    input  logic             rst_n,
    input  logic             push,
    input  logic             pop,
    input  logic [WIDTH-1:0] din,
    output logic [WIDTH-1:0] dout,
    output logic             full,
    output logic             empty,
    output logic [$clog2(DEPTH):0] count
);
```

### Cache Interface
```systemverilog
module cache #(
    parameter CACHE_SIZE = 16384,  // 16KB
    parameter LINE_SIZE  = 64,     // 64B line
    parameter WAYS       = 4       // 4-way set associative
)(
    input  logic        clk,
    input  logic        rst_n,
    // CPU interface
    input  logic        cpu_req,
    input  logic        cpu_wr,
    input  logic [31:0] cpu_addr,
    input  logic [31:0] cpu_wdata,
    output logic [31:0] cpu_rdata,
    output logic        cpu_ready,
    // Memory interface
    output logic        mem_req,
    output logic        mem_wr,
    output logic [31:0] mem_addr,
    output logic [LINE_SIZE*8-1:0] mem_wdata,
    input  logic [LINE_SIZE*8-1:0] mem_rdata,
    input  logic        mem_ready
);
```

---

## Common Patterns

### 1. Banked Memory for Bandwidth
```
4-bank memory:
┌─────────┬─────────┬─────────┬─────────┐
│ Bank 0  │ Bank 1  │ Bank 2  │ Bank 3  │
│ addr[1:0]=00 │ addr[1:0]=01 │ addr[1:0]=10 │ addr[1:0]=11 │
└─────────┴─────────┴─────────┴─────────┘
         │         │         │         │
         └────────┬┴─────────┴─────────┘
                  │
            Output MUX / Crossbar
```

### 2. Double-Buffering
```
Buffer A ──write──> [Processing] ──read──> Buffer B
    │                                         │
    └────────────< swap on frame >────────────┘
```

### 3. Read-Modify-Write
```
Cycle 1: Read old data from memory
Cycle 2: Modify (merge old with new)
Cycle 3: Write back merged data
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| FIFO pointer wraparound | Pointer comparison fails at wrap boundary | Use extra MSB bit for full/empty detection |
| Async FIFO binary pointer | Multi-bit change causes metastability | Use Gray code encoding (only 1 bit changes per increment) |
| Cache write-back race | New request corrupts line during write-back | Lock line during write-back, stall new requests |
| Read-during-write hazard | Undefined: old or new value returned? | Define behavior explicitly (write-first bypass or read-first) |
| SRAM timing mismatch | Read data not stable when captured | Add output register stage for timing closure |
| ECC single-bit correction delay | Correction adds to read latency | Pipeline ECC check; use bypass for non-correctable reads |
| Power-of-2 depth assumption | Non-power-of-2 FIFO depth causes pointer bugs | Use modular arithmetic or restrict depth to power-of-2 |

---

## Checklist

- [ ] Capacity meets requirements
- [ ] Bandwidth calculated and verified
- [ ] Latency within budget
- [ ] Error protection (ECC/parity) if needed
- [ ] Clock domain crossings handled
- [ ] Power gating considered
- [ ] BIST/MBIST hooks added
- [ ] Timing closure feasible
- [ ] Area within budget

---

## Resources

- `references/cache-design.md` - Cache architecture patterns
- `references/fifo-design.md` - FIFO implementation guide
- `references/memory-controller.md` *(planned)* - DDR/SRAM controllers
- `references/ecc-parity.md` *(planned)* - Error protection schemes
- `examples/example-l1-cache.md` *(planned)* - L1 cache implementation
- `examples/example-async-fifo.md` - CDC FIFO design
