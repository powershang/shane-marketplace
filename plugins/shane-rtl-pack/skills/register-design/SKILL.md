---
name: register-design
description: >-
  This skill handles CSR (Control/Status Register) and register file design
  including access types (RW, RO, W1C, RC, WO), address decoding, byte strobes,
  field extraction, and register generation. Use for "design register file",
  "create CSR block", "implement APB registers", "register map design",
  "add control registers", or any register/CSR design task.
---

# Register Design

## Quick Mode
When user asks for quick register help, focus on these 2 checks only:
1. **Access type correctness** — Verify W1C/RC/WO fields implement correct HW-set vs SW-clear priority
2. **Byte strobe handling** — Confirm partial writes only affect the targeted byte lanes

## When to Use
- Designing control/status register blocks
- Implementing memory-mapped configuration interfaces
- Creating register files for processor/peripheral interaction
- Generating RTL from register specifications

---

## Quick Reference

### Register Access Types

| Type | Name | Software | Hardware | Description |
|------|------|----------|----------|-------------|
| RW | Read-Write | R/W | - | Standard read-write |
| RO | Read-Only | R | W | Hardware writes, SW reads |
| WO | Write-Only | W | R | SW writes, HW reads |
| W1C | Write-1-Clear | W1C | Set | Write 1 to clear bit |
| W1S | Write-1-Set | W1S | - | Write 1 to set bit |
| W0C | Write-0-Clear | W0C | Set | Write 0 to clear bit |
| RC | Read-Clear | R | Set | Read clears register |
| RS | Read-Set | R | - | Read sets register |
| WC | Write-Clear | W | Set | Any write clears |
| RWAC | RW Auto-Clear | R/W | - | Auto-clears after write |

### Common Register Patterns

| Pattern | Use Case | Example |
|---------|----------|---------|
| Control | Start operations | `CTRL.START`, `CTRL.ENABLE` |
| Status | Report state | `STATUS.BUSY`, `STATUS.DONE` |
| Interrupt Enable | Mask interrupts | `IRQ_EN.TX_DONE_EN` |
| Interrupt Status | Pending interrupts | `IRQ_STATUS.TX_DONE` (W1C) |
| Configuration | Set parameters | `CONFIG.MODE`, `CONFIG.SPEED` |
| Data | Transfer data | `TX_DATA`, `RX_DATA` |
| Counter | Statistics | `PKT_COUNT`, `ERR_COUNT` |

---

## Methodology

### Step 1: Define Register Map

```
Offset  Name         Access  Reset     Description
------  -----------  ------  --------  -----------
0x00    CTRL         RW      0x0000    Control register
0x04    STATUS       RO      0x0000    Status register
0x08    IRQ_EN       RW      0x0000    Interrupt enable
0x0C    IRQ_STATUS   W1C     0x0000    Interrupt status
0x10    CONFIG       RW      0x0001    Configuration
0x14    TX_DATA      WO      -         Transmit data
0x18    RX_DATA      RO      -         Receive data
0x1C    VERSION      RO      0x0100    IP version
```

### Step 2: Define Field Layout

```
CTRL Register (0x00)
  [0]     START    RW    Start operation (auto-clear)
  [1]     ENABLE   RW    Enable module
  [3:2]   MODE     RW    Operating mode (0-3)
  [7:4]   Reserved -     -
  [15:8]  PRESCALE RW    Clock prescaler
  [31:16] Reserved -     -

STATUS Register (0x04)
  [0]     BUSY     RO    Operation in progress
  [1]     DONE     RO    Operation complete
  [2]     ERROR    RO    Error occurred
  [3]     FIFO_EMPTY RO  TX FIFO empty
  [4]     FIFO_FULL  RO  TX FIFO full
  [31:5]  Reserved -     -
```

### Step 3: Implement Address Decoder

```systemverilog
// Parameterized address decoder
module reg_decoder #(
  parameter ADDR_W = 12,
  parameter NUM_REGS = 8
)(
  input  logic [ADDR_W-1:0]   addr,
  output logic [NUM_REGS-1:0] reg_sel,
  output logic                valid
);

  always_comb begin
    reg_sel = '0;
    valid = 1'b0;

    if (addr[ADDR_W-1:$clog2(NUM_REGS)+2] == '0) begin
      reg_sel[addr[$clog2(NUM_REGS)+1:2]] = 1'b1;
      valid = 1'b1;
    end
  end

endmodule
```

### Step 4: Implement Register Storage

```systemverilog
// RW Register with byte strobes
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    reg_ctrl <= CTRL_RESET_VAL;
  end else if (wr_en && sel_ctrl) begin
    if (wr_strb[0]) reg_ctrl[7:0]   <= wr_data[7:0];
    if (wr_strb[1]) reg_ctrl[15:8]  <= wr_data[15:8];
    if (wr_strb[2]) reg_ctrl[23:16] <= wr_data[23:16];
    if (wr_strb[3]) reg_ctrl[31:24] <= wr_data[31:24];
  end
end

// W1C Register (interrupt status)
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    reg_irq_status <= '0;
  end else begin
    // Hardware sets bits
    if (hw_tx_done)  reg_irq_status[0] <= 1'b1;
    if (hw_rx_ready) reg_irq_status[1] <= 1'b1;
    if (hw_error)    reg_irq_status[2] <= 1'b1;

    // Software clears bits by writing 1
    if (wr_en && sel_irq_status) begin
      reg_irq_status <= reg_irq_status & ~wr_data;
    end
  end
end

// RC Register (read-clear counter)
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    reg_counter <= '0;
  end else if (rd_en && sel_counter) begin
    reg_counter <= '0;  // Clear on read
  end else if (count_event) begin
    reg_counter <= reg_counter + 1;
  end
end

// Auto-clear register (pulse generation)
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    reg_start <= 1'b0;
  end else if (wr_en && sel_ctrl && wr_data[0]) begin
    reg_start <= 1'b1;  // Set on write
  end else begin
    reg_start <= 1'b0;  // Auto-clear next cycle
  end
end
```

### Step 5: Implement Read Multiplexer

```systemverilog
always_comb begin
  rd_data = 32'h0;

  case (1'b1)
    sel_ctrl:       rd_data = reg_ctrl;
    sel_status:     rd_data = {27'h0, fifo_full, fifo_empty,
                               error, done, busy};
    sel_irq_en:     rd_data = reg_irq_en;
    sel_irq_status: rd_data = reg_irq_status;
    sel_config:     rd_data = reg_config;
    sel_rx_data:    rd_data = fifo_rd_data;
    sel_version:    rd_data = VERSION;
    default:        rd_data = 32'hDEAD_BEEF;
  endcase
end
```

---

## Register Block Template

```systemverilog
module reg_block #(
  parameter ADDR_W = 12,
  parameter DATA_W = 32
)(
  input  logic                 clk,
  input  logic                 rst_n,

  // Register interface
  input  logic                 reg_wr,
  input  logic                 reg_rd,
  input  logic [ADDR_W-1:0]    reg_addr,
  input  logic [DATA_W-1:0]    reg_wdata,
  input  logic [DATA_W/8-1:0]  reg_wstrb,
  output logic [DATA_W-1:0]    reg_rdata,
  output logic                 reg_error,

  // Hardware interface - outputs (from registers)
  output logic                 hw_enable,
  output logic [1:0]           hw_mode,
  output logic [7:0]           hw_prescale,
  output logic                 hw_start_pulse,

  // Hardware interface - inputs (to registers)
  input  logic                 hw_busy,
  input  logic                 hw_done,
  input  logic                 hw_error_flag,
  input  logic [31:0]          hw_rx_data
);

  // Address offsets
  localparam ADDR_CTRL    = 12'h000;
  localparam ADDR_STATUS  = 12'h004;
  localparam ADDR_IRQ_EN  = 12'h008;
  localparam ADDR_IRQ_ST  = 12'h00C;
  localparam ADDR_CONFIG  = 12'h010;
  localparam ADDR_RX_DATA = 12'h018;
  localparam ADDR_VERSION = 12'h01C;

  // Register storage
  logic [31:0] r_ctrl;
  logic [31:0] r_irq_en;
  logic [31:0] r_irq_status;
  logic [31:0] r_config;

  // Constants
  localparam VERSION = 32'h0001_0000;

  // Address decode
  wire sel_ctrl    = (reg_addr == ADDR_CTRL);
  wire sel_status  = (reg_addr == ADDR_STATUS);
  wire sel_irq_en  = (reg_addr == ADDR_IRQ_EN);
  wire sel_irq_st  = (reg_addr == ADDR_IRQ_ST);
  wire sel_config  = (reg_addr == ADDR_CONFIG);
  wire sel_rx_data = (reg_addr == ADDR_RX_DATA);
  wire sel_version = (reg_addr == ADDR_VERSION);

  wire addr_valid = sel_ctrl | sel_status | sel_irq_en | sel_irq_st |
                    sel_config | sel_rx_data | sel_version;

  // Byte strobe helper
  function automatic logic [31:0] apply_wstrb(
    input logic [31:0] old_val,
    input logic [31:0] new_val,
    input logic [3:0]  strb
  );
    return {strb[3] ? new_val[31:24] : old_val[31:24],
            strb[2] ? new_val[23:16] : old_val[23:16],
            strb[1] ? new_val[15:8]  : old_val[15:8],
            strb[0] ? new_val[7:0]   : old_val[7:0]};
  endfunction

  // CTRL register (RW with auto-clear START bit)
  logic start_pending;

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      r_ctrl <= 32'h0;
      start_pending <= 1'b0;
    end else begin
      // Auto-clear START bit
      if (start_pending) begin
        r_ctrl[0] <= 1'b0;
        start_pending <= 1'b0;
      end

      // Write
      if (reg_wr && sel_ctrl) begin
        r_ctrl <= apply_wstrb(r_ctrl, reg_wdata, reg_wstrb);
        if (reg_wdata[0] && reg_wstrb[0]) start_pending <= 1'b1;
      end
    end
  end

  // IRQ_EN register (RW)
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      r_irq_en <= 32'h0;
    end else if (reg_wr && sel_irq_en) begin
      r_irq_en <= apply_wstrb(r_irq_en, reg_wdata, reg_wstrb);
    end
  end

  // IRQ_STATUS register (W1C)
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      r_irq_status <= 32'h0;
    end else begin
      // HW sets
      if (hw_done)       r_irq_status[0] <= 1'b1;
      if (hw_error_flag) r_irq_status[1] <= 1'b1;

      // SW clears (W1C)
      if (reg_wr && sel_irq_st) begin
        r_irq_status <= r_irq_status &
                        ~apply_wstrb(32'h0, reg_wdata, reg_wstrb);
      end
    end
  end

  // CONFIG register (RW)
  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      r_config <= 32'h0000_0001;  // Default config
    end else if (reg_wr && sel_config) begin
      r_config <= apply_wstrb(r_config, reg_wdata, reg_wstrb);
    end
  end

  // Read mux
  always_comb begin
    case (1'b1)
      sel_ctrl:    reg_rdata = r_ctrl;
      sel_status:  reg_rdata = {29'h0, hw_error_flag, hw_done, hw_busy};
      sel_irq_en:  reg_rdata = r_irq_en;
      sel_irq_st:  reg_rdata = r_irq_status;
      sel_config:  reg_rdata = r_config;
      sel_rx_data: reg_rdata = hw_rx_data;
      sel_version: reg_rdata = VERSION;
      default:     reg_rdata = 32'hDEAD_BEEF;
    endcase
  end

  // Error on invalid address
  assign reg_error = (reg_wr || reg_rd) && !addr_valid;

  // Hardware outputs
  assign hw_enable      = r_ctrl[1];
  assign hw_mode        = r_ctrl[3:2];
  assign hw_prescale    = r_ctrl[15:8];
  assign hw_start_pulse = start_pending;

endmodule
```

---

## Field Extraction Macros

```systemverilog
// Define field positions
`define CTRL_START      0
`define CTRL_ENABLE     1
`define CTRL_MODE       3:2
`define CTRL_PRESCALE   15:8

// Extract field
wire start  = reg_ctrl[`CTRL_START];
wire enable = reg_ctrl[`CTRL_ENABLE];
wire [1:0] mode = reg_ctrl[`CTRL_MODE];
wire [7:0] prescale = reg_ctrl[`CTRL_PRESCALE];

// Alternative: packed struct (SystemVerilog)
typedef struct packed {
  logic [15:0] reserved1;
  logic [7:0]  prescale;
  logic [3:0]  reserved0;
  logic [1:0]  mode;
  logic        enable;
  logic        start;
} ctrl_reg_t;

ctrl_reg_t ctrl;
assign ctrl = ctrl_reg_t'(reg_ctrl);

// Use fields directly
wire start  = ctrl.start;
wire enable = ctrl.enable;
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| W1C race condition | HW set and SW clear happen same cycle, event lost | Give HW set higher priority; SW clears only already-set bits |
| Byte strobe ignored | 8-bit write overwrites entire 32-bit register | Implement per-byte write enable using `wr_strb` |
| RC without capture | Event during read-clear cycle is lost | Capture new event even during read: `counter <= event ? 1 : '0` |
| Auto-clear timing | Pulse clears same cycle as set, HW misses it | Use pending register for one-full-cycle pulse guarantee |
| Address decode gap | Unmapped address returns X or hangs bus | Return default (0 or error) for all unmapped addresses |
| Missing RO enforcement | Software writes to read-only field silently ignored | Document RO behavior; optionally flag write-to-RO as error |
| Reset value mismatch | RTL reset value differs from spec | Auto-generate reset values from register spec (IP-XACT/SystemRDL) |

---

## Verification Checklist

- [ ] All registers accessible at documented offsets
- [ ] Reset values match specification
- [ ] RW registers read back written value
- [ ] RO registers ignore writes
- [ ] W1C bits clear correctly
- [ ] RC registers clear on read
- [ ] Auto-clear bits pulse for one cycle
- [ ] Byte strobes work correctly
- [ ] Invalid addresses return error/default
- [ ] Hardware inputs reflected in status registers
- [ ] Hardware outputs driven from control registers

---

## References

- `references/csr-patterns.md` - Register access type patterns
- `references/register-gen.md` - Register generation from spec
- `references/apb-regfile.md` - APB interface register file

## Examples

- `examples/example-csr-block.md` - Complete CSR block example
