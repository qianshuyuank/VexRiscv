# Murax SoC 工程实践指南

Murax 是一个基于 VexRiscv 核心的轻量级 SoC，旨在提供一个极简的 RISC-V 运行环境，非常适合在资源受限的 FPGA（如 iCE40）上运行。本文档详细介绍了 `Murax.v` 的使用方法、硬件组成及软件开发流程。

## 1. 引脚定义 (Pin Definitions)

`Murax.v` 顶层模块的引脚如下：

| 引脚名称 | 类型 | 描述 |
| :--- | :--- | :--- |
| `io_asyncReset` | Input | 异步复位信号，高电平有效 |
| `io_mainClk` | Input | 主时钟输入 |
| `io_jtag_tms` | Input | JTAG 状态切换 |
| `io_jtag_tdi` | Input | JTAG 数据输入 |
| `io_jtag_tdo` | Output | JTAG 数据输出 |
| `io_jtag_tck` | Input | JTAG 时钟 |
| `io_gpioA_read[31:0]` | Input | GPIO A 输入数据 |
| `io_gpioA_write[31:0]` | Output | GPIO A 输出数据 |
| `io_gpioA_writeEnable[31:0]` | Output | GPIO A 输出使能（三态控制） |
| `io_uart_txd` | Output | UART 发送端 |
| `io_uart_rxd` | Input | UART 接收端 |

## 2. 模块功能组成 (Functional Composition)

Murax SoC 内部集成了以下核心组件：

- **VexRiscv CPU**: 采用 RV32I[M] 指令集，带有 5 级流水线（取决于具体生成配置）。
- **On-Chip RAM**: 默认 8 KB（可配置），用于存储指令和数据。
- **Interconnect**: 采用 Pipelined Memory Bus 作为主总线，通过 APB3 桥连接外设。
- **Peripherals**:
    - **GPIO A**: 32 位通用 IO，支持读写和三态控制。
    - **UART**: 带有硬件发送/接收 FIFO 的串口。
    - **Timers**: 包含一个 16 位预分频器（Prescaler）和两个 16 位定时器（Timer A, Timer B）。
- **JTAG Debugger**: 支持通过 OpenOCD 进行在线调试。

## 3. 内存映射 (Memory Map)

软件开发时需根据以下地址空间访问硬件资源：

| 资源名称 | 起始地址 | 范围/大小 | 说明 |
| :--- | :--- | :--- | :--- |
| **RAM** | `0x80000000` | 8 KB+ | 默认复位向量指向此处 |
| **GPIO A** | `0xF0000000` | 4 KB | 偏移 0x00000 |
| **UART** | `0xF0010000` | 4 KB | 偏移 0x10000 |
| **Timer Prescaler** | `0xF0020000` | - | 定时器预分频配置 |
| **Timer Interrupt** | `0xF0020010` | - | 定时器中断控制器 |
| **Timer A** | `0xF0020040` | - | 定时器 A 计数/配置 |
| **Timer B** | `0xF0020050` | - | 定时器 B 计数/配置 |

## 4. 程序加载 (Program Loading)

Murax 支持两种主要的程序加载方式：

### 4.1 静态加载 (Hex 文件初始化)
在生成 Verilog 时，可以将编译好的 `.hex` 文件直接映射到片上 RAM 中。
- **文件路径**: `src/main/ressource/hex/muraxDemo.hex` (默认示例)
- **实现方式**: 在 Scala 定义中使用 `onChipRamHexFile` 参数，生成的 Verilog 会包含 `$readmemh` 指令。

### 4.2 动态加载 (JTAG/OpenOCD)
通过 JTAG 接口在运行时将程序下载到 RAM 中：
1. 使用交叉编译工具链（如 `riscv64-unknown-elf-gcc`）编译生成 `.elf` 文件。
2. 启动 OpenOCD 连接硬件。
3. 使用 GDB 命令 `load` 将程序写入 `0x80000000`。

## 5. 调试说明 (Debugging)

Murax 的调试体系基于 JTAG 协议，兼容标准 OpenOCD。

### 5.1 连接 OpenOCD
你需要一个支持 JTAG 的调试器（如 FT2232H, J-Link, ST-Link 等）。
启动命令示例：
```bash
openocd -f tcl/interface/your_jtag_adapter.cfg -f tcl/target/murax.cfg
```

### 5.2 使用 GDB 调试
1. 启动 `riscv64-unknown-elf-gdb`。
2. 连接到 OpenOCD：
   ```gdb
   target remote localhost:3333
   ```
3. 加载程序并运行：
   ```gdb
   load your_program.elf
   continue
   ```

### 5.3 仿真调试
若无硬件，可使用 Verilator 进行仿真。
- 仿真路径: `src/test/cpp/murax`
- 执行: `make clean run`
- 仿真过程中支持通过 TCP 暴露 JTAG 端口，OpenOCD 可通过 `jtag_tcp.cfg` 接入。

---
*注：Murax 的源代码位于 [Murax.scala](file:///e:/github/VexRiscv/src/main/scala/vexriscv/demo/Murax.scala)。*
