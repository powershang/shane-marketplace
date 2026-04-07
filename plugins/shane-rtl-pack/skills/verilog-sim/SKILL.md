---
name: verilog-sim
description: Use this skill whenever the user wants to simulate, run, test, or verify Verilog (.v) code. Triggers include "跑 simulation", "驗證", "run verilog", "test verilog", "simulate", or any request to execute Verilog code and see output. Uses local Icarus Verilog via WSL.
allowed-tools: Bash, Write, Read
---

# Verilog Simulation (Local Icarus Verilog + WSL)

## Environment Setup

- **WSL**: Ubuntu-24.04 已安裝 Icarus Verilog 12.0 + GTKWave v3.3.116
- 安裝方式：`sudo apt install iverilog gtkwave` (已在 WSL 內執行過)

### Path 問題（重要！）

直接用 Git Bash 的 `wsl` 命令會有 path 轉換問題。**務必用以下方式**：

```powershell
# 透過 powershell.exe 調用 wsl
powershell.exe -Command "wsl 你的命令"
```

### PowerShell 命令連結

- **只能用 `;`**，不能用 `&&`（&& 是 Bash 語法，PowerShell 不支援）

```powershell
# 錯誤
powershell.exe -Command "wsl iverilog ... && wsl vvp ..."

# 正確 - 用分號
powershell.exe -Command "wsl iverilog -g2012 -o /tmp/tb.vvp /mnt/c/你的路徑/design.sv; wsl vvp /tmp/tb.vvp"

# 或者分開執行
powershell.exe -Command "wsl iverilog -g2012 -o /tmp/tb.vvp /mnt/c/你的路徑/design.sv"
powershell.exe -Command "wsl vvp /tmp/tb.vvp"
```

## 使用命令

### 1. 編譯 (SystemVerilog 2012)
```powershell
powershell.exe -Command "wsl iverilog -g2012 -o /tmp/design.vvp /mnt/c/你的路徑/design.v /mnt/c/你的路徑/testbench.v"
```

### 2. 執行 Simulation
```powershell
powershell.exe -Command "wsl vvp /tmp/design.vvp"
```
- 波形檔會產生在 testbench 設定的 `$dumpfile("wave.vcd")` 路徑

### 3. 查看波形
```powershell
powershell.exe -Command "wsl gtkwave /mnt/c/你的路徑/wave.vcd"
```

### 路徑對應
- Windows: `C:\Users\shane_wu\project`
- WSL: `/mnt/c/Users/shane_wu/project`

## GTKWave Alias 設定

在 GTKWave 中新增 alias（View → Aliases）：
```
top0=top.uut0
top1=top.uut1
top2=top.uut2
top3=top.uut3
bot0=bot.uut0
bot1=bot.uut1
bot2=bot.uut2
bot3=bot.uut3
dut_state=top.state
dut_phase_cnt=top.phase_cnt
```

## Testbench 範本

```verilog
`timescale 1ns/1ps

module tb;
    // 訊號宣告
    reg clk;
    reg rst_n;
    wire [7:0] data;

    // 時脈產生 (100MHz)
    initial begin
        clk = 0;
        forever #5 clk = ~clk;
    end

    // DUT 實例
    dut uut (.*);

    // 波形輸出
    initial begin
        $dumpfile("wave.vcd");
        $dumpvars(0, tb);
    end

    // 測試序列
    initial begin
        rst_n = 0;
        #20;
        rst_n = 1;
        #100;
        $display("Done. Final: %d", data);
        $finish;
    end
endmodule
```

## 快速別名（可設為 shell alias）

在 Windows PowerShell 或 Git Bash 加入：
```powershell
# 編譯
function iverilog-build {
    $files = $args -join " "
    powershell.exe -Command "wsl iverilog -g2012 -o /tmp/design.vvp $files"
}
# 執行
function vvp-run {
    powershell.exe -Command "wsl vvp /tmp/design.vvp"
}
# 查看波形
function gtkwave-show {
    powershell.exe -Command "wsl gtkwave /mnt/c/$args/wave.vcd"
}
```
