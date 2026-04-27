## 目录

- [目录](#目录)
- [描述](#描述)
- [面积使用率和最高频率](#面积使用率和最高频率)
- [依赖项](#依赖项)
- [CPU生成](#cpu生成)
- [回归测试](#回归测试)
- [通过GDB OpenOCD和Verilator对模拟CPU进行交互式调试](#通过gdb-openocd和verilator对模拟cpu进行交互式调试)
- [使用Eclipse运行和调试软件](#使用eclipse运行和调试软件)
  - [使用gnu-mcu-eclipse](#使用gnu-mcu-eclipse)
  - [使用Zylin插件（旧版）](#使用zylin插件旧版)
- [Briey SoC](#briey-soc)
- [Murax SoC](#murax-soc)
- [运行Linux](#运行linux)
- [构建RISC-V GCC](#构建risc-v-gcc)
- [CPU参数化和实例化示例](#cpu参数化和实例化示例)
- [通过插件系统向CPU添加自定义指令](#通过插件系统向cpu添加自定义指令)
- [通过插件系统添加新的CSR](#通过插件系统添加新的csr)
- [CPU时钟和复位](#cpu时钟和复位)
- [VexRiscv架构](#vexriscv架构)
  - [FPU](#fpu)
  - [插件](#插件)
    - [IBusSimplePlugin](#ibussimpleplugin)
    - [IBusCachedPlugin](#ibuscachedplugin)
    - [DecoderSimplePlugin](#decodersimpleplugin)
    - [RegFilePlugin](#regfileplugin)
    - [HazardSimplePlugin](#hazardsimpleplugin)
    - [SrcPlugin](#srcplugin)
    - [IntAluPlugin](#intaluplugin)
    - [LightShifterPlugin](#lightshifterplugin)
    - [FullBarrelShifterPlugin](#fullbarrelshifterplugin)
    - [BranchPlugin](#branchplugin)
      - [预测 NONE](#预测-none)
      - [预测 STATIC](#预测-static)
      - [预测 DYNAMIC](#预测-dynamic)
      - [预测 DYNAMIC_TARGET](#预测-dynamic_target)
    - [DBusSimplePlugin](#dbussimpleplugin)
    - [DBusCachedPlugin](#dbuscachedplugin)
    - [MulPlugin](#mulplugin)
    - [DivPlugin](#divplugin)
    - [MulDivIterativePlugin](#muldiviterativeplugin)
    - [CsrPlugin](#csrplugin)
    - [MstatushPlugin](#mstatushplugin)
    - [StaticMemoryTranslatorPlugin](#staticmemorytranslatorplugin)
    - [MmuPlugin](#mmuplugin)
    - [PmpPlugin](#pmpplugin)
      - [PmpPluginNapot](#pmppluginnapot)
    - [DebugPlugin](#debugplugin)
    - [EmbeddedRiscvJtag](#embeddedRiscvJtag)
    - [YamlPlugin](#yamlplugin)
    - [FpuPlugin](#fpuplugin)
    - [AesPlugin](#aesplugin)



## 描述

本仓库存放的是使用SpinalHDL编写的RISC-V实现。以下是一些规格：

- RV32I[M][A][F[D]][C]指令集
- 流水线从2到5+级（[取指*X]，解码，执行，[内存]，[回写]）
- 当启用几乎所有功能时为1.44 DMIPS/MHz --no-inline（启用除法查找表时为1.57 DMIPS/MHz）
- 针对FPGA优化，不使用任何供应商特定的IP块/原语
- 支持AXI4、Avalon、wishbone
- 可选的MUL/DIV扩展
- 可选的F32/F64 FPU（目前需要数据缓存）
- 可选的指令和数据缓存
- 可选的硬件重填MMU
- 可选的调试扩展，允许通过GDB >> openOCD >> JTAG连接进行Eclipse调试
- 可选的中断和异常处理，支持[RISC-V特权ISA规范v1.10](https://riscv.org/specifications/privileged-isa/)中定义的Machine、[Supervisor]和[User]模式
- 两种移位指令实现：单周期（全桶形移位器）和shiftNumber周期
- 每个阶段都可以有可选的旁路或互锁 hazard 逻辑
- 兼容Linux（SoC：https://github.com/enjoy-digital/linux-on-litex-vexriscv）
- 兼容Zephyr
- [FreeRTOS移植](https://github.com/Dolu1990/FreeRTOS-RISCV)
- 支持I$ D$上的紧密耦合内存（参见GenFullWithTcm / GenFullWithTcmIntegrated）

该CPU的硬件描述采用了一种非常软件导向的方法（在生成的硬件中没有任何开销）。以下是使用的软件概念列表：

- 固定的东西很少。几乎所有东西都是基于插件的。PC管理器是一个插件，寄存器文件是一个插件，hazard控制器是一个插件，...
- 有一个自动化工具，允许插件在给定阶段向流水线插入数据，并允许其他插件在另一阶段通过自动流水线读取它。
- 有一个服务系统，提供了一个非常动态的框架。例如，一个插件可以提供一个异常服务，其他插件可以使用该服务从流水线发出异常。

有一个gitter频道用于解答关于VexRiscv的所有问题：<br>
[![Gitter](https://badges.gitter.im/SpinalHDL/VexRiscv.svg)](https://gitter.im/SpinalHDL/VexRiscv?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

如需商业支持，请联系spinalhdl@gmail.com。

注意，您可能对VexiiRiscv感兴趣（https://github.com/SpinalHDL/VexiiRiscv）。

## 面积使用率和最高频率

以下数字是通过将CPU作为顶层综合在最快速度等级上获得的，没有使用任何特定综合选项来节省面积或获得更好的最大频率（中性）。<br>
时钟约束设置为一个无法达到的值，这倾向于增加设计面积。<br>
dhrystone基准测试使用`-O3 -fno-inline`选项编译。<br>
所有带缓存的配置在dhrystone基准测试期间都会有一些缓存抖动，除了`VexRiscv full max perf`配置。这当然会降低性能。可以生成适合4KB I$和4KB D$的dhrystone二进制文件（我曾经有过这种情况），但目前还不是这样。<br>
下面使用的CPU配置可以在`src/scala/vexriscv/demo`目录中找到。

```
VexRiscv small (RV32I, 0.52 DMIPS/MHz, 无数据路径旁路, 无中断) ->
    Artix 7     -> 243 MHz 504 LUT 505 FF 
    Cyclone V   -> 174 MHz 352 ALMs
    Cyclone IV  -> 179 MHz 731 LUT 494 FF 
    iCE40       -> 92 MHz 1130 LC

VexRiscv small (RV32I, 0.52 DMIPS/MHz, 无数据路径旁路) ->
    Artix 7     -> 240 MHz 556 LUT 566 FF 
    Cyclone V   -> 194 MHz 394 ALMs
    Cyclone IV  -> 174 MHz 831 LUT 555 FF 
    iCE40       -> 85 MHz 1292 LC

VexRiscv small and productive (RV32I, 0.82 DMIPS/MHz)  ->
    Artix 7     -> 232 MHz 816 LUT 534 FF 
    Cyclone V   -> 155 MHz 492 ALMs
    Cyclone IV  -> 155 MHz 1,111 LUT 530 FF 
    iCE40       -> 63 MHz 1596 LC

VexRiscv small and productive with I$ (RV32I, 0.70 DMIPS/MHz, 4KB-I$)  ->
    Artix 7     -> 220 MHz 730 LUT 570 FF 
    Cyclone V   -> 142 MHz 501 ALMs
    Cyclone IV  -> 150 MHz 1,139 LUT 536 FF 
    iCE40       -> 66 MHz 1680 LC

VexRiscv full no cache (RV32IM, 1.21 DMIPS/MHz 2.30 Coremark/MHz, 单周期桶形移位器, 调试模块, 捕获异常, 静态分支) ->
    Artix 7     -> 216 MHz 1418 LUT 949 FF 
    Cyclone V   -> 133 MHz 933 ALMs
    Cyclone IV  -> 143 MHz 2,076 LUT 972 FF 

VexRiscv full (RV32IM, 1.21 DMIPS/MHz 2.30 Coremark/MHz 带缓存抖动, 4KB-I$,4KB-D$, 单周期桶形移位器, 调试模块, 捕获异常, 静态分支) ->
    Artix 7     -> 199 MHz 1840 LUT 1158 FF 
    Cyclone V   -> 141 MHz 1,166 ALMs
    Cyclone IV  -> 131 MHz 2,407 LUT 1,067 FF 

VexRiscv full max perf (HZ*IPC) -> (RV32IM, 1.38 DMIPS/MHz 2.57 Coremark/MHz, 8KB-I$,8KB-D$, 单周期桶形移位器, 调试模块, 捕获异常, 在取指阶段动态分支预测, 分支和移位操作在执行阶段完成) ->
    Artix 7     -> 200 MHz 1935 LUT 1216 FF 
    Cyclone V   -> 130 MHz 1,166 ALMs
    Cyclone IV  -> 126 MHz 2,484 LUT 1,120 FF 

VexRiscv full with MMU (RV32IM, 1.24 DMIPS/MHz 2.35 Coremark/MHz, 带缓存抖动, 4KB-I$, 4KB-D$, 单周期桶形移位器, 调试模块, 捕获异常, 动态分支, MMU) ->
    Artix 7     -> 151 MHz 2021 LUT 1541 FF 
    Cyclone V   -> 124 MHz 1,368 ALMs
    Cyclone IV -> 128 MHz 2,826 LUT 1,474 FF 

VexRiscv linux balanced (RV32IMA, 1.21 DMIPS/MHz 2.27 Coremark/MHz, 带缓存抖动, 4KB-I$, 4KB-D$, 单周期桶形移位器, 捕获异常, 静态分支, MMU, Supervisor, 兼容主流linux) ->
    Artix 7     -> 180 MHz 2883 LUT 2130 FF 
    Cyclone V   -> 131 MHz 1,764 ALMs
    Cyclone IV  -> 121 MHz 3,608 LUT 2,082 FF 
```

以下配置可达到1.44 DMIPS/MHz：

- 5级：F -> D -> E -> M -> WB
- 单周期ADD/SUB/位运算/移位ALU
- 分支/跳转在E阶段完成
- 内存加载值在WB阶段旁路（延迟结果）
- 在M阶段进行33周期除法并旁路（延迟结果）
- 在WB阶段进行单周期乘法并旁路（延迟结果）
- 在F阶段使用直接映射目标缓冲缓存进行动态分支预测（正确预测时无惩罚）

注意，最近添加了移除取指/内存/回写阶段的能力，以减少CPU面积，这导致更小的CPU和更好的小配置DMIPS/MHz。

## 依赖项

在Ubuntu 14上：

```sh
# JAVA JDK 8
sudo add-apt-repository -y ppa:openjdk-r/ppa
sudo apt-get update
sudo apt-get install openjdk-8-jdk -y
sudo update-alternatives --config java
sudo update-alternatives --config javac

# 安装SBT - https://www.scala-sbt.org/
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo apt-key add
sudo apt-get update
sudo apt-get install sbt

# Verilator（仅用于仿真，确实需要3.9+，通常apt-get会给你3.8）
sudo apt-get install git make autoconf g++ flex bison
git clone http://git.veripool.org/git/verilator   # 仅第一次
unsetenv VERILATOR_ROOT  # 对于csh；如果在bash上忽略错误
unset VERILATOR_ROOT  # 对于bash
cd verilator
git pull        # 确保我们是最新的
git checkout v4.216
autoconf        # 创建./configure脚本
./configure
make
sudo make install
```

## CPU生成
我们现在有二十二种CPU配置[在此目录中](./src/main/scala/vexriscv/demo)。查看名为Gen*.scala的文件。这是[完整配置](./src/main/scala/vexriscv/demo/GenFull.scala)，这是[最小配置](./src/main/scala/vexriscv/demo/GenSmallest.scala)。 

要生成对应的RTL作为`VexRiscv.v`文件，在本仓库的根目录下运行以下命令：

```sh
sbt "runMain vexriscv.demo.GenFull"
```

或

```sh
sbt "runMain vexriscv.demo.GenSmallest"
```

注意：
- 第一次运行可能需要一些时间。
- VexRiscv项目可能需要SpinalHDL仓库的未发布master-head版本。如果编译失败，只需获取SpinalHDL仓库并在其中执行"sbt clean compile publishLocal"，如依赖项章节中所述。

## 回归测试

[![Build Status](https://travis-ci.org/SpinalHDL/VexRiscv.svg?branch=master)](https://travis-ci.org/SpinalHDL/VexRiscv)

要运行测试（需要java、scala、verilator），只需执行：

```sh
export VEXRISCV_REGRESSION_SEED=42
export VEXRISCV_REGRESSION_TEST_ID=
sbt "testOnly vexriscv.TestIndividualFeatures"
```

这将生成随机VexRiscv配置并用以下测试进行测试： 
- 来自https://github.com/riscv/riscv-tests/tree/master/isa和https://github.com/riscv/riscv-compliance的ISA测试
- Dhrystone基准测试
- Coremark基准测试
- Zephyr os
- Buildroot/Linux os
- 一些手写测试来检查CSR、调试模块和MMU插件

您可以通过设置VEXRISCV_REGRESSION_TEST_ID来重新运行一些特定测试，用它们的id。例如，如果您想重新运行：
- test_id_5_test_IBus_CachedS1024W1BPL32Relaxvexriscv.plugin.DYNAMIC_DBus_CachedS8192W2BPL16_MulDiv_MulDivFpga_Shift_FullLate_Branch_Late_Hazard_BypassAll_RegFile_SyncDR_Src__Csr_AllNoException_Decoder__Debug_None_DBus_NoMmu
- test_id_9_test_IBus_Simple1S2InjStagevexriscv.plugin.STATIC_DBus_SimpleLate_MulDiv_MulDivFpgaSimple_Shift_FullEarly_Branch_Late_Hazard_Interlock_RegFile_AsyncER_Src_AddSubExecute_Csr_None_Decoder__Debug_None_DBus_NoMmu

然后：

```
export VEXRISCV_REGRESSION_TEST_ID=5,9
```

还有一些环境变量可用于调节随机生成：

| 参数                                  | 范围              | 描述 |
| ------------------------------------------- | ------------------ | ----------- |
| VEXRISCV_REGRESSION_SEED                    | Int                | 用于生成随机配置的种子 |        
| VEXRISCV_REGRESSION_TEST_ID                 | \[Int\[,\Int\]\*\] | 应保留并测试的随机配置 |                        
| VEXRISCV_REGRESSION_CONFIG_COUNT            | Int                | 随机配置的数量 |                        
| VEXRISCV_REGRESSION_CONFIG_RVC_RATE         | 0.0-1.0            | 生成RVC配置的机会 |                               
| VEXRISCV_REGRESSION_CONFIG_LINUX_RATE       | 0.0-1.0            | 生成linux就绪配置的机会 |            
| VEXRISCV_REGRESSION_CONFIG_MACHINE_OS_RATE  | 0.0-1.0            | 生成机器模式OS就绪配置的机会 |            
| VEXRISCV_REGRESSION_LINUX_REGRESSION        | yes/no             | 启用linux测试 |           
| VEXRISCV_REGRESSION_COREMARK                | yes/no             | 启用Coremark测试 |           
| VEXRISCV_REGRESSION_ZEPHYR_COUNT            | Int                | 在兼容配置上运行的zephyr测��数量 |        
| VEXRISCV_REGRESSION_CONFIG_DEMW_RATE        | 0.0-1.0            | 生成带回写阶段配置的机会 |            
| VEXRISCV_REGRESSION_CONFIG_DEM_RATE         | 0.0-1.0            | 生成带内存阶段配置的机会 |            

## 基本Verilator仿真

要运行基本仿真并使用stdout且不追踪，直接加载二进制文件受`src/test/cpp/regression/makefile`的`RUN_HEX`变量支持。这比使用GDB over OpenOCD over JTAG over TCP有显著的性能优势。VCD追踪受makefile变量`TRACE`支持。

## 通过GDB OpenOCD和Verilator对模拟CPU进行交互式调试

要使用此功能，您只需使用与运行测试相同的命令，但在make参数中添加`DEBUG_PLUGIN_EXTERNAL=yes`。
这适用于`GenFull`配置，但不适用于`GenSmallest`，因为该配置没有调试模块。

然后，您可以使用[OpenOCD RISC-V](https://github.com/SpinalHDL/openocd_riscv)工具创建连接到目标（模拟CPU）的GDB服务器，如下所示：

```sh
#在VexRiscv仓库中，运行可以连接OpenOCD的仿真 =>
sbt "runMain vexriscv.demo.GenFull"
cd src/test/cpp/regression
make run DEBUG_PLUGIN_EXTERNAL=yes

#在openocd git中，构建之后 =>
src/openocd -c "set VEXRISCV_YAML PATH_TO_THE_GENERATED_CPU0_YAML_FILE" -f tcl/target/vexriscv_sim.cfg

#使用RISC-V可执行文件（GenFull CPU）运行GDB会话
YourRiscvToolsPath/bin/riscv32-unknown-elf-gdb VexRiscvRepo/src/test/resources/elf/uart.elf
target remote localhost:3333
monitor reset halt
load
continue

# 现在它应该在CPU的Verilator仿真中打印消息
```

## 使用Eclipse运行和调试软件

### 使用gnu-mcu-eclipse

您可以从这里下载该IDE的版本：https://github.com/gnu-mcu-eclipse/org.eclipse.epp.packages/releases

在IDE中，您可以通过以下方式导入makefile项目：
- file -> import -> C/C++ -> existing Code as Makefile Project
- 选择包含makefile的文件夹，然后选择"Cross GCC"（不是"RISC-V Cross GCC"）

要创建新的调试配置：
- run -> Debug Configurations -> GDB OpenOCD Debugging 双击
- 查看https://drive.google.com/open?id=1c46tyEV0xLwOsk76b0y2qqs8CYy7Zq3f获取配置示例

### 使用Zylin插件（旧版）
您可以使用Eclipse + Zylin embedded CDT插件来完成（http://opensource.zylin.com/embeddedcdt.html）。已在Helios Service Release 2（http://www.Eclipse.org/downloads/download.php?file=/technology/epp/downloads/release/helios/SR2/Eclipse-cpp-helios-SR2-linux-gtk-x86_64.tar.gz）和相应的zylin插件上进行了测试。

以下命令将下载Eclipse并安装插件。

```sh
wget http://www.eclipse.org/downloads/download.php?file=/technology/epp/downloads/release/helios/SR2/eclipse-cpp-helios-SR2-linux-gtk-x86_64.tar.gz
tar -xvzf download.php?file=%2Ftechnology%2Fepp%2Fdownloads%2Frelease%2Fhelios%2FSR2%2Feclipse-cpp-helios-SR2-linux-gtk-x86_64.tar.gz
cd eclipse
./eclipse -application org.eclipse.equinox.p2.director -repository http://opensource.zylin.com/zylincdt -installIU com.zylin.cdt.feature.feature.group/
```

查看https://drive.google.com/drive/folders/1NseNHH05B6lmIXqQFVwK8xRjWE4ydeG-?usp=sharing导入makefile项目并创建调试配置。

注意，有时需要重启Eclipse才能放置新的断点。

如果您想获取更多关于JTAG / GDB工作原理的信息，您可以在这里找到很棒的博客：

https://tomverbeure.github.io/2021/07/18/VexRiscv-OpenOCD-and-Traps.html

## Briey SoC
作为演示，一个名为Briey的SoC在`src/main/scala/vexriscv/demo/Briey.scala`中实现。这个SoC与[Pinsec SoC](https://spinalhdl.github.io/SpinalDoc-RTD/v1.3.1/SpinalHDL/Legacy/pinsec/hardware_toplevel.html)非常相似：

![Briey SoC](assets/brieySoc.png?raw=true "")

要生成Briey SoC硬件：

```sh
sbt "runMain vexriscv.demo.Briey"
```

要运行Briey SoC的verilator仿真，然后可以连接到OpenOCD/GDB，首先获取这些依赖项：

```sh
sudo apt-get install build-essential xorg-dev libudev-dev libgl1-mesa-dev libglu1-mesa-dev libasound2-dev libpulse-dev libopenal-dev libogg-dev libvorbis-dev libaudiofile-dev libpng12-dev libfreetype6-dev libusb-dev libdbus-1-dev zlib1g-dev libdirectfb-dev libsdl2-dev
```

然后进入`src/test/cpp/briey`并运行仿真（UART TX打印在终端，VGA显示在GUI中）：

```sh
make clean run
```

要连接OpenOCD（https://github.com/SpinalHDL/openocd_riscv）到仿真：

```sh
src/openocd -f tcl/interface/jtag_tcp.cfg -c "set BRIEY_CPU0_YAML /home/spinalvm/Spinal/VexRiscv/cpu0.yaml" -f tcl/target/briey.cfg
```
要连接OpenOCD到Altera FPGA（Intel VJTAG）参见：https://github.com/SpinalHDL/VexRiscv/tree/master/doc/vjtag

您可以在此找到多个软件示例和演示：https://github.com/SpinalHDL/VexRiscvSocSoftware/tree/master/projects/briey

您可以在此找到一些实例化Briey SoC的FPGA项目（DE1-SoC, DE0-Nano）：https://drive.google.com/drive/folders/0B-CqLXDTaMbKZGdJZlZ5THAxRTQ?usp=sharing


以下是Briey SoC的时序和面积测量：

```
Artix 7     -> 181 MHz 3220 LUT 3181 FF 
Cyclone V   -> 142 MHz 2,222 ALMs
Cyclone IV  -> 130 MHz 4,538 LUT 3,211 FF 
```

## Murax SoC

Murax是一个非常轻量级的SoC（可以装入ICE40 FPGA），可以在没有任何外部组件的情况下工作：
- VexRiscv RV32I[M]
- JTAG调试器（Eclipse/GDB/openocd就绪）
- 8 kB片上RAM
- 中断支持
- 外设的APB总线
- 32个GPIO引脚
- 一个16位预分频器，两个16位定时器
- 一个带tx/rx FIFO的UART

根据CPU配置，在使用icestorm综合的ICE40-hx8k FPGA上，完整SoC具有以下面积/性能：
- RV32I互锁级 => 51 MHz, 2387 LC 0.45 DMIPS/MHz
- RV32I旁路级 => 45 MHz, 2718 LC 0.65 DMIPS/MHz

其实现可在`src/main/scala/vexriscv/demo/Murax.scala`中找到。

要生成Murax SoC硬件：

```sh
# 生成RAM中没有任何内容的SoC
sbt "runMain vexriscv.demo.Murax"

# 生成RAM中已包含演示程序的SoC
sbt "runMain vexriscv.demo.MuraxWithRamInit"
```

`MuraxWithRamInit`默认包含的演示程序将闪烁LED并回显通过UART接收到的字符。要在运行Verilator仿真时看到此效果，请输入一些文本并按Enter。

然后进入`src/test/cpp/murax`并运行仿真：

```sh
make clean run
```

要连接OpenOCD（https://github.com/SpinalHDL/openocd_riscv）到仿真：

```sh
src/openocd -f tcl/interface/jtag_tcp.cfg -c "set MURAX_CPU0_YAML /home/spinalvm/Spinal/VexRiscv/cpu0.yaml" -f tcl/target/murax.cfg
```

您可以在此找到多个软件示例和演示：https://github.com/SpinalHDL/VexRiscvSocSoftware/tree/master/projects/murax

以下是Murax SoC的一些时序和面积测量：

```
Murax interlocked stages (0.45 DMIPS/MHz, 8 bits GPIO) ->
    Artix 7     -> 216 MHz 1109 LUT 1201 FF 
    Cyclone V   -> 182 MHz 725 ALMs
    Cyclone IV  -> 147 MHz 1,551 LUT 1,223 FF 
    iCE40       ->  64 MHz 2422 LC (nextpnr)

MuraxFast bypassed stages (0.65 DMIPS/MHz, 8 bits GPIO) ->
    Artix 7     -> 224 MHz 1278 LUT 1300 FF 
    Cyclone V   -> 173 MHz 867 ALMs
    Cyclone IV  -> 143 MHz 1,755 LUT 1,258 FF 
    iCE40       ->  66 MHz 2799 LC (nextpnr)
```

一些用于生成SoC和调用icestorm工具链的脚本可在`scripts/Murax/`中找到。

使用SpinalSim实现了一个具有相同功能+ GUI的顶级仿真测试台。您可以在`src/test/scala/vexriscv/MuraxSim.scala`中找到它。

要运行它：

```sh
# 这将生成Murax RTL + 运行其测试台。您需要安装Verilator 3.9xx。
sbt "test:runMain vexriscv.MuraxSim"
```

## 使用mill构建以上所有内容

Mill是一个简单的构建Scala/Java的工具，在离线环境中也非常适合。

Github地址：https://github.com/com-lihaoyi/mill

文档：https://mill-build.com/mill/Intro_to_Mill.html

下载可执行文件mill： 

```sh
curl --fail -L -o mill https://github.com/com-lihaoyi/mill/releases/download/0.11.6/0.11.6-assembly
chmod +x mill
```
使用mill生成对应的RTL作为`VexRiscv.v`文件，在本仓库的根目录下运行以下命令：

```sh
./mill VexRiscv.runMain vexriscv.demo.GenFull
```
或

```sh
./mill VexRiscv.runMain vexriscv.demo.GenSmallest
```

使用mill运行测试（需要java、scala、verilator）：

```sh
export VEXRISCV_REGRESSION_SEED=42
export VEXRISCV_REGRESSION_TEST_ID=
./mill VexRiscv.test.testOnly vexriscv.TestIndividualFeatures
```

使用mill生成Briey SoC硬件：

```sh
./mill VexRiscv.runMain vexriscv.demo.Briey
```

使用mill生成Murax SoC硬件：

```sh
# 生成RAM中没有任何内容的SoC
./mill VexRiscv.runMain vexriscv.demo.Murax

# 生成RAM中已包含演示程序的SoC
./mill VexRiscv.runMain vexriscv.demo.MuraxWithRamInit

# 这将生成Murax RTL + 运行其测试台。您需要安装Verilator 3.9xx。
./mill VexRiscv.test.runMain vexriscv.MuraxSim
```

Mill的IDE支持：

```sh
# 构建服务器协议（BSP）
./mill mill.bsp.BSP/install

# IntelliJ IDEA支持
./mill mill.idea.GenIdea/idea
```


## 运行Linux

默认配置位于`src/main/scala/vexriscv/demo/Linux.scala`。

该文件还包含
- 编译buildroot镜像的命令
- 如何以交互模式运行Verilator仿真

目前没有SoC可以在硬件上运行它，这是WIP。但CPU仿真已经可以启动linux并运行用户空间应用程序（包括python）。

注意，VexRiscv可以在带缓存和不带缓存的设计上运行Linux。

## 构建RISC-V GCC

预构建的GCC工具套件可在以下位置找到：

- https://www.sifive.com/software/ => 预构建的RISC-V GCC工具链和模拟器

VexRiscvSocSoftware makefile期望在/opt/riscv/__contentOfThisPreBuild__中找到Sifive GCC工具链。

您可以通过https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14.tar.gz手动下载它

```sh
# 下载并安装Sifive GCC工具链
version=riscv64-unknown-elf-gcc-8.3.0-2019.08.0-x86_64-linux-ubuntu14
wget -O riscv64-unknown-elf-gcc.tar.gz riscv https://static.dev.sifive.com/dev-tools/$version.tar.gz
tar -xzvf riscv64-unknown-elf-gcc.tar.gz
sudo mv $version /opt/riscv
echo 'export PATH=/opt/riscv/bin:$PATH' >> ~/.bashrc
```

如果您想从源代码编译rv32i和rv32im GCC工具链并安装到`/opt/`，请执行以下操作（需要一小时）：

```sh
# 注意，有时git clone在成功克隆riscv-gnu-toolchain时会出问题。
sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev -y

git clone --recursive https://github.com/riscv/riscv-gnu-toolchain riscv-gnu-toolchain
cd riscv-gnu-toolchain

echo "Starting RISC-V Toolchain build process"

ARCH=rv32im
rmdir -rf $ARCH
mkdir $ARCH; cd $ARCH
../configure  --prefix=/opt/$ARCH --with-arch=$ARCH --with-abi=ilp32
sudo make -j4
cd ..


ARCH=rv32i
rmdir -rf $ARCH
mkdir $ARCH; cd $ARCH
../configure  --prefix=/opt/$ARCH --with-arch=$ARCH --with-abi=ilp32
sudo make -j4
cd ..

echo -e "\\nRISC-V Toolchain installation completed!"
```

## CPU参数化和实例化示例

您可以在https://github.com/SpinalHDL/VexRiscv/tree/master/src/main/scala/vexriscv/demo文件夹中找到许多不同配置的示例。

以下是其中一个示例：

```scala
import vexriscv._
import vexriscv.plugin._

//实例化一个VexRiscv
val cpu = new VexRiscv(
  //提供一个配置实例
  config = VexRiscvConfig(
    //提供一个插件列表，这些插件将把自己的逻辑添加到CPU中
    plugins = List(
      new IBusSimplePlugin(
        resetVector = 0x00000000l,
        cmdForkOnSecondStage = true,
        cmdForkPersistence  = true
      ),
      new DBusSimplePlugin(
        catchAddressMisaligned = false,
        catchAccessFault = false
      ),
      new DecoderSimplePlugin(
        catchIllegalInstruction = false
      ),
      new RegFilePlugin(
        regFileReadyKind = Plugin.SYNC,
        zeroBoot = true
      ),
      new IntAluPlugin,
      new SrcPlugin(
        separatedAddSub = false,
        executeInsertion = false
      ),
      new LightShifterPlugin,
      new HazardSimplePlugin(
        bypassExecute           = false,
        bypassMemory            = false,
        bypassWriteBack         = false,
        bypassWriteBackBuffer   = false
      ),
      new BranchPlugin(
        earlyBranch = false,
        catchAddressMisaligned = false
      ),
      new YamlPlugin("cpu0.yaml")
    )
  )
)
```

## 通过插件系统向CPU添加自定义指令

以下是一个简单插件的示例，该插件添加了一个简单的SIMD_ADD指令：

```scala
import spinal.core._
import vexriscv.plugin.Plugin
import vexriscv.{Stageable, DecoderService, VexRiscv}

//此插件示例将添加一个名为SIMD_ADD的新指令，执行以下操作：
//
//RD : 寄存器文���目标, RS : 寄存器文件源
//RD( 7 downto  0) = RS1( 7 downto  0) + RS2( 7 downto  0)
//RD(16 downto  8) = RS1(16 downto  8) + RS2(16 downto  8)
//RD(23 downto 16) = RS1(23 downto 16) + RS2(23 downto 16)
//RD(31 downto 24) = RS1(31 downto 24) + RS2(31 downto 24)
//
//指令编码：
//0000011----------000-----0110011
//       |RS2||RS1|   |RD |
//
//注意：RS1、RS2、RD位置遵循RISC-V规范，对ISA的所有指令都是通用的

class SimdAddPlugin extends Plugin[VexRiscv]{
  //定义IS_SIMD_ADD信号的概念，该信号指定当前指令是否针对此插件
  object IS_SIMD_ADD extends Stageable(Bool)

  //设置插件并请求不同服务的回调
  override def setup(pipeline: VexRiscv): Unit = {
    import pipeline.config._

    //获取DecoderService实例
    val decoderService = pipeline.service(classOf[DecoderService])

    //当指令被解码时指定IS_SIMD_ADD的默认值
    decoderService.addDefault(IS_SIMD_ADD, False)

    //当指令匹配'key'模式时指定应应用的指令解码
    decoderService.add(
      //新SIMD_ADD指令的位模式
      key = M"0000011----------000-----0110011",

      //当识别���'key'模式时的解码规范
      List(
        IS_SIMD_ADD              -> True,
        REGFILE_WRITE_VALID      -> True, //启用寄存器文件写
        BYPASSABLE_EXECUTE_STAGE -> True, //通知hazard管理单元指令结果已在执行阶段可访问（旁路就绪）
        BYPASSABLE_MEMORY_STAGE  -> True, //同上，但针对内存阶段
        RS1_USE                  -> True, //通知hazard管理单元此指令使用RS1值
        RS2_USE                  -> True  //同上，但针对RS2
      )
    )
  }

  override def build(pipeline: VexRiscv): Unit = {
    import pipeline._
    import pipeline.config._

    //在执行阶段添加一个新的作用域（用于给信号命名）
    execute plug new Area {
      //定义插件内部使用的一些信号
      val rs1 = execute.input(RS1).asUInt
      //regfile[RS1]的32位UInt值
      val rs2 = execute.input(RS2).asUInt
      val rd = UInt(32 bits)

      //执行一些计算
      rd(7 downto 0) := rs1(7 downto 0) + rs2(7 downto 0)
      rd(16 downto 8) := rs1(16 downto 8) + rs2(16 downto 8)
      rd(23 downto 16) := rs1(23 downto 16) + rs2(23 downto 16)
      rd(31 downto 24) := rs1(31 downto 24) + rs2(31 downto 24)

      //当指令是SIMD_ADD时，将结果写入寄存器文件数据路径
      when(execute.input(IS_SIMD_ADD)) {
        execute.output(REGFILE_WRITE_DATA) := rd.asBits
      }
    }
  }
}
```

如果您想将此插件添加到给定CPU，只需将其添加到其参数化插件列表中。

这个例子非常简单，但每个插件都可以真正访问整个CPU：
- 暂停CPU的给定阶段
- 取消调度指令
- 发出异常
- 引入新的指令解码规范
- 请求将PC跳转到某处
- 读取其他插件发布的信号
- 覆盖发布的信号值
- 提供替代实现
- ...

作为演示，此SimdAddPlugin已集成在`src/main/scala/vexriscv/demo/GenCustomSimdAdd.scala` CPU配置中，并通过`src/test/cpp/custom/simd_add`应用程序进行自测，运行以下命令：

```sh
# 生成CPU
sbt "runMain vexriscv.demo.GenCustomSimdAdd"

cd src/test/cpp/regression/

# 可选择添加TRACE=yes如果您想从仿真中获取VCD波形。
# 还要注意，默认情况下，测试台会引入指令/数据总线停顿。
# 注意CUSTOM_SIMD_ADD标志设置为yes。
make clean run IBUS=SIMPLE DBUS=SIMPLE CSR=no MMU=no DEBUG_PLUGIN=no MUL=no DIV=no DHRYSTONE=no REDO=2 CUSTOM_SIMD_ADD=yes
```

要在波形查看器中检索插件相关信号，只需用`simd`过滤。

## 通过插件系统添加新的CSR

以下是关于如何通过插件系统向CPU添加自定义CSR的两个示例：
https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/CustomCsrDemoPlugin.scala

第一个（`CustomCsrDemoPlugin`）将指令计数器和时钟周期计数器添加到CSR映射中（并作为演示做一些棘手的事情）。

第二个（`CustomCsrDemoGpioPlugin`）创建一个直接映射到CSR的GPIO外设。

## CPU时钟和复位

如果没有调试插件，CPU将有一个标准的`clk`输入和一个`reset`输入。但使用调试插件情况如下：

- `clk`：与之前一样，驱动整个CPU设计的时钟，包括调试逻辑
- `reset`：复位所有CPU状态，除了调试逻辑
- `debugReset`：复位CPU的调试逻辑
- `debug_resetOut`：CPU输出信号，允许JTAG复位CPU +内存互连+外设

以下是使用调试插件时的复位互连：

```
                                VexRiscv
                            +------------------+
                            |                  |
toplevelReset >----+--------> debugReset       |
                   |        |                  |
                   |  +-----< debug_resetOut   |
                   |  |     |                  |
                   +--or>-+-> reset            |
                          | |                  |
                          | +------------------+
                          |
                          +-> Interconnect / Peripherals
```


## VexRiscv架构

VexRiscv通过5级顺序流水线实现，许多可选且互补的插件添加功能以提供功能完整的RISC-V CPU。
这种方法完全不传统，只有通过元硬件描述语言（目前是SpinalHDL）才可能实现，但通过VexRiscv实现证明了其优势：
- 您可以直接通过插件系统交换/打开/关闭CPU的各个部分
- 您可以添加新功能/指令而无需修改CPU的任何源代码
- 它允许CPU配置覆盖非常广泛的实现范围，而不会导致意大利面条式代码
- 它允许您的代码库真正生成参数化CPU设计

如果您生成不带任何插件的CPU，它将只包含5个流水线阶段及其基本仲裁的定义，但仅此而已，
其他所有内容（包括程序计数器）都通过插件添加到CPU中。

### FPU

特性：

- 支持IEEE 754浮点和可选的双精度
- 实现次正规数（次正规数加载/存储情况下会损失几个周期）
- 实现异常标志
- FPU可以在多个CPU之间共享
- 可通过FpuPlugin集成在CPU内部或外部
- 完全流水线化，对于大多数操作（加、减、乘、fma、加载、存储），只要没有相互依赖，可以每周期产生一个结果
- 使用多个子乘法操作的并行实现乘法（"FPGA友好"）
- 除法用radix 4实现（每周期2位）
- 平方根用radix 2实现（每周期1位）
- 目前仅与DBusCachedPlugin兼容用于加载和存储
- 64位加载和存储可以通过DBusCachedPlugin在一个周期内完成（即使VexRiscv是RV32）

精度、舍入（RNE、RTZ、RDN、RUP、RMM）和合规性：

- 完全实现，除了下面指定的情况
- 在FMA中，乘法结果在加法之前舍入（保持尾数宽度+2位）
- 一个非常特殊的下溢标志情况不遵循IEEE 754（从次正规数舍入到正规数）
- 非常特殊的情况下，SGNJ指令不会改变F32/F64的值（没有NaN-boxing变异）
 
 有一个FPU设计及其CPU集成的图表：
 
 ![fpuDesign](assets/fpuDesign.png?raw=true "")
 
 FPU可以使用FpuParameter数据结构进行参数化：
 
 | 参数 | 类型 | 描述 |
 | ------ | ----------- | ------ |
 | withDouble   | Boolean | 启用64位浮点（32位始终启用） |
 | asyncRegFile   | Boolean | 使用组合读实现寄存器文件（而不是同步读） |
 | mulWidthA   | Boolean | 指定乘法块左操作数的宽度 |
 | mulWidthB   | Boolean | 同上，但针对右操作数 |

FPU本身的综合结果，不包括CPU集成，在快速速度等级上：

```
Fpu 32 bits ->
  Artix 7 relaxed -> 135 MHz 1786 LUT 1778 FF 
  Artix 7 FMax    -> 205 MHz 2101 LUT 1778 FF 
Fpu 64/32 bits ->
  Artix 7 relaxed -> 101 MHz 3336 LUT 3033 FF 
  Artix 7 FMax    -> 165 MHz 3728 LUT 3175 FF 
```

注意，如果您想通过openocd_riscv.vexriscv目标调试FPU代码，您需要使用来自的GDB：

https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-20171231-x86_64-linux-centos6.tar.gz

更新版本的gdb将无法检测到FPU。此外，openocd_riscv.vexriscv无法读取CSR/FPU寄存器，因此要对浮点值有可见性，您需要使用-O0编译代码，这将强制将值存储在内存中（因此可见）

### 插件

本节描述当前实现的插件。

- [IBusSimplePlugin](#ibussimpleplugin)
- [IBusCachedPlugin](#ibuscachedplugin)
- [DecoderSimplePlugin](#decodersimpleplugin)
- [RegFilePlugin](#regfileplugin)
- [HazardSimplePlugin](#hazardsimpleplugin)
- [SrcPlugin](#srcplugin)
- [IntAluPlugin](#intaluplugin)
- [LightShifterPlugin](#lightshifterplugin)
- [FullBarrelShifterPlugin](#fullbarrelshifterplugin)
- [BranchPlugin](#branchplugin)
- [DBusSimplePlugin](#dbussimpleplugin)
- [DBusCachedPlugin](#dbuscachedplugin)
- [MulPlugin](#mulplugin)
- [DivPlugin](#divplugin)
- [MulDivIterativePlugin](#muldiviterativeplugin)
- [CsrPlugin](#csrplugin)
- [MstatushPlugin](#mstatushplugin)
- [StaticMemoryTranslatorPlugin](#staticmemorytranslatorplugin)
- [MemoryTranslatorPlugin](#memorytranslatorplugin)
- [DebugPlugin](#debugplugin)
- [EmbeddedRiscvJtag](#embeddedRiscvJtag)
- [YamlPlugin](#yamlplugin)
- [FpuPlugin](#fpuplugin)


#### IBusSimplePlugin

此插件通过一个非常简单的中性内存接口实现CPU前端（指���取指）。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| catchAccessFault | Boolean | 当为true时，带读错误指示的指令读响应会导致CPU异常陷阱。 |
| resetVector | BigInt | 复位后程序计数器的地址。 |
| cmdForkOnSecondStage | Boolean | 当为false时，分支立即更新程序计数器。这最小化了分支惩罚，但可能降低FMax，因为指令总线地址信号是一个组合路径。当为true时，移除此组合路径，并在检测到分支后一个周期更新程序计数器。虽然FMax可能改善，但也会产生额外的分支惩罚。 |
| cmdForkPersistence  | Boolean | 当为false时，iBus上的请求可以在被确认之前消失/更改。这减少了面积，但与许多仲裁/从设备不安全/不支持。当为true时，一旦发起，iBus请求将保持直到被确认。 |
| compressedGen | Boolean | 启用RISC-V压缩指令（RVC）支持。 |
| busLatencyMin | Int | 指定iBus.cmd和iBus.rsp之间的最小延迟。添加相应数量的阶段到前端以保持IPC为1。 |
| injectorStage | Boolean | 当为true时，在CPU前端和解码阶段之间添加一个阶段以改善FMax。（busLatencyMin + injectorStage）应该至少为二。 |
| prediction | BranchPrediction | 可设置为NONE/STATIC/DYNAMIC/DYNAMIC_TARGET以指定分支预测器实现。详见下文。 |
| historyRamSizeLog2 | Int | 指定DYNAMIC/DYNAMIC_TARGET实现的直接映射预测缓存中的条目数。2的historyRamSizeLog2次方个条目。 |

以下是SimpleBus接口定义：

```scala
case class IBusSimpleCmd() extends Bundle{
  val pc = UInt(32 bits)
}

case class IBusSimpleRsp() extends Bundle with IMasterSlave{
  val error = Bool
  val inst  = Bits(32 bits)

  override def asMaster(): Unit = {
    out(error,inst)
  }
}

case class IBusSimpleBus(interfaceKeepData : Boolean) extends Bundle with IMasterSlave{
  var cmd = Stream(IBusSimpleCmd())
  var rsp = Flow(IBusSimpleRsp())

  override def asMaster(): Unit = {
    master(cmd)
    slave(rsp)
  }
}
```

**重要**：检查cmdForkPersistence参数，因为如果未设置，它可能会破坏iBus与您内存系统的兼容性（除非您外部添加一些缓冲区）。

设置cmdForkPersistence和cmdForkOnSecondStage可改善iBus cmd时序。

iBusSimplePlugin包括桥接器，可将IBusSimpleBus转换为AXI4、Avalon和Wishbone接口。

此插件实现了一个跳转接口，允许所有其他插件发出跳转：

```scala
trait JumpService{
  def createJumpInterface(stage : Stage) : Flow[UInt]
}
```

stage参数指定发出跳转请求的阶段。这允许PcManagerSimplePlugin插件管理来自不同阶段的跳转请求的优先级。

#### IBusCachedPlugin

简单轻量的多路指令缓存。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| resetVector | BigInt | 复位后程序计数器的地址。 |
| relaxedPcCalculation | Boolean | 当为false时，分支立即更新程序计数器。这最小化了分支惩罚，但可能降低FMax，因为指令总线地址信号是一个组合路径。当为true时，移除此组合路径，并在检测到分支后一个周期更新程序计数器。虽然FMax可能改善，但也会产生额外的分支惩罚。 |
| prediction | BranchPrediction | 可设置为NONE/STATIC/DYNAMIC/DYNAMIC_TARGET以指定分支预测器实现。详见下文。 |
| historyRamSizeLog2 | Int | 指定DYNAMIC/DYNAMIC_TARGET实现的直接映射预测缓存中的条目数。2的historyRamSizeLog2次方个条目 |
| compressedGen | Boolean | 启用RISC-V压缩指令（RVC）支持。 |
| config.cacheSize  | Int | 缓存的总存储容量（字节）。 |
| config.bytePerLine  | Int | 每缓存行的字节数  |
| config.wayCount  | Int | 缓存路数 |
| config.twoCycleRam  | Boolean | 在解码阶段而不是取指阶段检查标签值以放松时序 |
| config.asyncTagMemory  | Boolean | 以异步方式而不是同步方式读取缓存标签  |
| config.addressWidth  | Int | CPU地址宽度。应为32 |
| config.cpuDataWidth  | Int | CPU数据宽度。应为32 |
| config.memDataWidth  | Int | 内存数据宽度。可能不是32，但目前只测试了32 |
| config.catchIllegalAccess  | Boolean  | 捕获对无效内存地址（MMU）的内存访问 |
| config.catchAccessFault  | Boolean | 捕获内存总线响应错误时 |
| config.catchMemoryTranslationMiss  | Boolean  |  捕获MMU的TLB未命中 |

注意：如果启用twoCycleRam选项且wayCount大于1，则应将寄存器文件插件配置为以异步方式读取regFile。

内存总线定义如下：

```scala
case class InstructionCacheMemCmd(p : InstructionCacheConfig) extends Bundle{
  val address = UInt(p.addressWidth bit)
  val size = UInt(log2Up(log2Up(p.bytePerLine) + 1) bits)
}

case class InstructionCacheMemRsp(p : InstructionCacheConfig) extends Bundle{
  val data = Bits(p.memDataWidth bit)
  val error = Bool
}

case class InstructionCacheMemBus(p : InstructionCacheConfig) extends Bundle with IMasterSlave{
  val cmd = Stream (InstructionCacheMemCmd(p))
  val rsp = Flow (InstructionCacheMemRsp(p))

  override def asMaster(): Unit = {
    master(cmd)
    slave(rsp)
  }
}
```

地址以字节为单位，并与bytePerLine配置对齐，大小始终等于log2(bytePerLine)。

注意，cmd流事务需要在开始返回rsp事务之前被消费（最小1周期延迟）

有关Stream的一些文档：

https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?highlight=stream

Flow与Stream相同，但没有ready信号。


#### DecoderSimplePlugin

此插件为其他插件提供指令解码能力。

例如，对于给定指令，流水线hazard插件需要知道它是否使用寄存器文件源1/2，以便在hazard消失之前停止流水线。
每个实现指令的插件都向DecoderSimplePlugin插件提供这种信息。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| catchIllegalInstruction | Boolean | 当为true时，不匹配解码规范的指令将产生陷阱异常 |

以下是使用示例：

```scala
    //当指令匹配'key'模式时指定应应用的指令解码
    decoderService.add(
      //新指令的位模式
      key = M"0000011----------000-----0110011",

      //当识别出'key'模式时的解码规范
      List(
        IS_SIMD_ADD              -> True, //通知流水线当前指令是SIMD_ADD指令
        REGFILE_WRITE_VALID      -> True, //通知hazard管理单元此指令写入寄存器文件
        BYPASSABLE_EXECUTE_STAGE -> True, //通知hazard管理单元指令结果已在执行阶段可访问（旁路就绪）
        BYPASSABLE_MEMORY_STAGE  -> True, //同上，但针对内存阶段
        RS1_USE                  -> True, //通知hazard管理单元此指令使用RS1值
        RS2_USE                  -> True  //同上，但针对RS2
      )
    )
  }
```

此插件在解码阶段操作。

#### RegFilePlugin

此插件实现寄存器文件。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| regFileReadyKind | RegFileReadKind | 可设置为ASYNC或SYNC。指定用于实现寄存器文件的存储器读取类型。ASYNC表示零周期延迟存储器读取，而SYNC表示一周期延迟存储器读取，可以映射到标准FPGA存储器块   |
| zeroBoot | Boolean | 在仿真开始时将所有寄存器加载为零，以保持日志/追踪中的所有内容确定性 |

此寄存器文件使用"不在乎"读-写策略，因此旁路/hazard插件应处理这种情况。

如果您收到`Missing inserts : INSTRUCTION_ANTICIPATE`错误，那是因为RegFilePlugin配置为使用SYNC存储器读取端口访问寄存器文件，但IBus插件配置无法在解码阶段前一周期提供指令的寄存器文件读地址。解决方法是：

- 将RegFilePlugin配置为以异步方式（ASYNC）实现寄存器文件读，如果您目标设备支持的话
- 如果您使用IBusSimplePlugin，需要启用injectorStage配置
- 如果您使用IBusCachedPlugin，可以启用injectorStage，或将twoCycleCache + twoCycleRam设置为false。

#### HazardSimplePlugin

此插件检查流水线指令依赖关系，必要时或可能时，将在解码阶段停止指令或旁路来自后续阶段到解码阶段的指令结果。
由于寄存器文件使用"不在乎"读-写策略，此插件也管理这类hazard。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| bypassExecute | Boolean | 启用来自执行阶段指令结果的旁路 |
| bypassMemory | Boolean | 启用来自内存阶段指令结果的旁路 |
| bypassWriteBack | Boolean | 启用来自回写阶段指令结果的旁路 |
| bypassWriteBackBuffer | Boolean | 启用上一个周期寄存器文件写值的旁路  |

#### SrcPlugin

此插件将不同输入值进行多路复用，以产生SRC1/SRC2/SRC_ADD/SRC_SUB/SRC_LESS值，这些是执行阶段（ALU/分支/加载/存储）许多插件使用的常见值。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| separatedAddSub | RegFileReadKind | 默认情况下，SRC_ADD/SRC_SUB由单个可控加法器/减法器生成，但如果设置为true，则使用单独的加法器/减法器 |
| executeInsertion | Boolean | 默认情况下，SRC1/SRC2在解码阶段生成，但如果此参数为true，则在执行阶段完成（将放松旁路网络） |

除SRC1/SRC2外，此插件在执行阶段开始时完成所有操作。

#### IntAluPlugin

此插件使用SrcPlugin输出在执行阶段实现所有ADD/SUB/SLT/SLTU/XOR/OR/AND/LUI/AUIPC指令。这是一个非常简单的插件。

结果直接在执行阶段结束时注入流水线。

#### LightShifterPlugin

使用迭代移位寄存器实现SLL/SRL/SRA指令，每位移位使用一个周期。

结果直接在执行阶段结束时注入流水线。

#### FullBarrelShifterPlugin

使用全桶形移位器实现SLL/SRL/SRA指令，因此在一个周期内完成所有移位。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| earlyInjection | Boolean | 默认情况下，移位结果在内存阶段注入流水线以放松时序，但如果此选项为true，则在执行阶段完成 |

#### BranchPlugin

此插件实现所有分支/跳转指令（JAL/JALR/BEQ/BNE/BLT/BGE/BLTU/BGEU），并提供CPU前端插件用于实现分支预测的基元。预测实现在前端插件（IBusX）中设置。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| earlyBranch | Boolean | 默认情况下，分支在内存阶段完成以放松时序，但如果设置此选项，则在执行阶段完成|
| catchAddressMisaligned | Boolean | 如果在未对齐的PC地址执行跳转/分支，将触发陷阱异常 |

每次预测错误的跳转将产生2到4周期的惩罚，取决于`earlyBranch`和`PcManagerSimplePlugin.relaxedPcCalculation`配置

##### 预测 NONE

无预测：每次由跳转/分支引起的PC更改都将产生惩罚。

##### 预测 STATIC

在解码阶段，向后条件分支或JAL被推测分支。如果推测正确，分支惩罚减少到单个周期，
否则应用标准惩罚。

##### 预测 DYNAMIC

与STATIC预测相同，只是为了进行预测，它使用直接映射的2位历史缓存（BHT），该缓存记住分支是否更可能被执行。

##### 预测 DYNAMIC_TARGET

此预测器在取指阶段使用直接映射分支目标缓冲（BTB），存储指令的PC、指令的目标PC和2位历史以记住分支是否更可能被执行。这实际上是VexRiscv上实现的最有效的分支预测器，因为当分支预测正确时，它不会产生分支惩罚。
缺点是此预测器有一个很长的组合路径，从预测缓存读取端口到程序计数器，通过跳转接口。

#### DBusSimplePlugin

此插件通过简单的内存总线实现加载和存储指令（LB/LH/LW/LBU/LHU/LWU/SB/SH/SW）。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| catchAddressMisaligned | Boolean | 如果访问未对齐的内存地址，将触发陷阱异常 |
| catchAccessFault | Boolean | 如果内存读返回错误，将触发陷阱异常  |
| earlyInjection | Boolean | 默认情况下，内存读值在回写阶段注入流水线以放松时序。如果此参数为true，则在内存阶段完成 |

以下是DBusSimpleBus

```scala
case class DBusSimpleCmd() extends Bundle{
  val wr = Bool
  val address = UInt(32 bits)
  val data = Bits(32 bit)
  val size = UInt(2 bit)
}

case class DBusSimpleRsp() extends Bundle with IMasterSlave{
  val ready = Bool
  val error = Bool
  val data = Bits(32 bit)

  override def asMaster(): Unit = {
    out(ready,error,data)
  }
}


case class DBusSimpleBus() extends Bundle with IMasterSlave{
  val cmd = Stream(DBusSimpleCmd())
  val rsp = DBusSimpleRsp()

  override def asMaster(): Unit = {
    master(cmd)
    slave(rsp)
  }
}
```

注意有可用的桥接器可以将此接口转换为AXI4和Avalon。

cmd和相应的rsp之间至少有一个周期延迟。读cmd后，在rsp出现之前rsp.ready标志应为false。

#### DBusCachedPlugin

多路缓存实现，采用写直达和读时分配策略。（文档WIP）

您可以通过0x500F指令使整个缓存失效，您可以通过指令0x500F | RS1 << 15使地址范围（单行大小）失效，其中RS1不应为X0，指向要失效地址的某个字节。


内存总线定义如下：

```scala
case class DataCacheMemCmd(p : DataCacheConfig) extends Bundle{
  val wr = Bool
  val uncached = Bool
  val address = UInt(p.addressWidth bit)
  val data = Bits(p.cpuDataWidth bits)
  val mask = Bits(p.cpuDataWidth/8 bits)
  val size   = UInt(p.sizeWidth bits) //... 1 => 2 bytes ... 2 => 4 bytes ...
  val exclusive = p.withExclusive generate Bool()
  val last = Bool
}
case class DataCacheMemRsp(p : DataCacheConfig) extends Bundle{
  val aggregated = UInt(p.aggregationWidth bits)
  val last = Bool()
  val data = Bits(p.memDataWidth bit)
  val error = Bool
  val exclusive = p.withExclusive generate Bool()
}
case class DataCacheInv(p : DataCacheConfig) extends Bundle{
  val enable = Bool()
  val address = UInt(p.addressWidth bit)
}
case class DataCacheAck(p : DataCacheConfig) extends Bundle{
  val hit = Bool()
}

case class DataCacheSync(p : DataCacheConfig) extends Bundle{
  val aggregated = UInt(p.aggregationWidth bits)
}

case class DataCacheMemBus(p : DataCacheConfig) extends Bundle with IMasterSlave{
  val cmd = Stream (DataCacheMemCmd(p))
  val rsp = Flow (DataCacheMemRsp(p))

  val inv = p.withInvalidate generate Stream(Fragment(DataCacheInv(p)))
  val ack = p.withInvalidate generate Stream(Fragment(DataCacheAck(p)))
  val sync = p.withInvalidate generate Stream(DataCacheSync(p))

  override def asMaster(): Unit = {
    master(cmd)
    slave(rsp)

    if(p.withInvalidate) {
      slave(inv)
      master(ack)
      slave(sync)
    }
  }
}
```

如果您不使用内存一致性，可以忽略inv/ack/sync流，写cmd不应产生任何rsp事务。

由于缓存是写直达的，没有写突发，只有单独的写事务。

地址以字节为单位，并与bytePerLine配置对齐，大小编码为log2(突发中的字节数)。
last只应在突发的最后一个事务上设置。

注意，cmd流事务需要在开始返回rsp事务之前被消费（最小1周期延迟）

有关Stream的一些文档：

https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?highlight=stream

Flow与Stream相同，但没有ready信号。

#### MulPlugin

实现RISC-V M扩展的乘法指令。其实现使用4个17*17位乘法以FPGA友好的方式完成。
处理在执行/内存/回写阶段之间完全流水线化。指令结果始终插入回写阶段。

#### DivPlugin

实现RISC-V M扩展的除法/模指令。以简单的迭代方式完成，始终需要34个周期。结果插入内存阶段。

此插件现在基于MulDivIterativePlugin。

#### MulDivIterativePlugin

此插件以迭代方式实现RISC-V M扩展的乘法、除法和模运算，这对没有DSP块的小型FPGA很友好。

此插件可以展开迭代计算过程，以减少执行mul/div指令所使用的周期数。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| genMul    | Boolean | 启用乘法支持。如果想改用MulPlugin，可设置为false |
| genDiv    | Boolean |  启用除法支持 |
| mulUnrollFactor    | Int | 用于加速乘法的组合阶段数，应> 0 |
| divUnrollFactor    | Int | 用于加速除法的组合阶段数，应> 0 |

执行乘法所用的周期数是'32/mulUnrollFactor'
执行除法所用的周期数是'32/divUnrollFactor + 1'

乘法和除法都在内存阶段处理（延迟结果）。

#### CsrPlugin

实现RISC-V特权规范中指定的大部分Machine模式寄存器和少量User模式寄存器。
大多数CSR的访问模式是参数化的，以减少不需要功能的面积使用。

（CsrAccess可以是`NONE/READ_ONLY/WRITE_ONLY/READ_WRITE`）

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| catchIllegalAccess   | Boolean |  |
| mvendorid            | BigInt |  |
| marchid              | BigInt |  |
| mimpid               | BigInt |  |
| mhartid              | BigInt |  |
| misaExtensionsInit   | Int |  |
| misaAccess           | CsrAccess |  |
| mtvecAccess          | CsrAccess |  |
| mtvecInit            | BigInt |  |
| mepcAccess           | CsrAccess |  |
| mscratchGen          | Boolean |  |
| mcauseAccess         | CsrAccess |  |
| mbadaddrAccess       | CsrAccess |  |
| mcycleAccess         | CsrAccess |  |
| minstretAccess       | CsrAccess |  |
| ucycleAccess         | CsrAccess |  |
| wfiGen               | Boolean |  |
| ecallGen             | Boolean |  |

如果发生中断，在跳转到mtvec之前，插件将停止预取阶段并等待后续流水线阶段中的所有指令完成执行。

如果发生异常，插件将终止相应指令，冲洗所有前面的指令，并等待直到被终止的指令到达回写阶段再跳转到mtvec。

#### MstatushPlugin

实现RV32系统的mstatush CSR（0x310）。此CSR表示概念性64位mstatus寄存器的上半部分32位。

对于VexRiscv（RV32，小端，无超级管理程序扩展），mstatush硬连线为零。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| readOnly | Boolean | 当为true（默认）时，mstatush是只读的。当为false时，接受写入但没有效果。 |

此插件是可选的。只有在需要mstatush CSR支持时才将其包含在配置中。

使用示例：
```scala
// 在VexRiscv配置插件列表中：
new MstatushPlugin(readOnly = true)  // 只读（默认）
new MstatushPlugin(readOnly = false) // 允许写入（无效果）
```

参见[GenWithMstatush.scala](./src/main/scala/vexriscv/demo/GenWithMstatush.scala)获取完整示例。

#### StaticMemoryTranslatorPlugin

静态内存转换插件，允许指定内存地址的哪个范围是I/O映射的，不应该被缓存。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| ioRange   | UInt => Bool | 函数引用，接受一个地址，如果该地址应该不被缓存则返回true。例如：ioRange= _(31 downto 28) === 0xF => 所有0xFXXXXXXX将不被缓存|


#### MmuPlugin

硬件重填MMU实现。允许其他插件（如DBusCachedPlugin/IBusCachedPlugin）实例化内存地址转换端口。每个端口都有一个小型的专用全关联TLB缓存，通过dbus访问共享自动重填。

#### PmpPlugin

这是物理内存保护（PMP）插件，符合v1.12 RISC-V特权规范，不支持ePMP（`Smepmp`）扩展。PMP通过写入两个特殊CSR配置：`pmpcfg#`和`pmpaddr#`。前者包含四个保护区域的权限和寻址模式，后者包含单个区域的编码起始地址。由于实际区域边界必须从写入这些寄存器的值计算，写入需要几个CPU周期。此延迟是必要的，以便将所有解码逻辑集中到一个组件中。否则，即使解码操作仅在重新编程PMP时发生（例如在某些上下文切换时），也必须为每个区域复制解码逻辑。

##### PmpPluginNapot

`PmpPluginNapot`是专门的PMP实现，仅提供`NAPOT`（自然对齐的2次幂区域）寻址模式。与完整的`PmpPlugin`相比，它需要更少的资源，并且对时序的影响更小。

#### DebugPlugin

此插件实现了足够的CPU调试功能，以允许舒适的GDB/Eclipse调试。为了访问这些调试功能，它提供了一个简单的内存总线接口。
JTAG接口由另一个桥接器提供，这使得可以将多个CPU有效地连接到同一个JTAG。

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| debugClockDomain   | ClockDomain | 由于调试单元能够自行复位CPU，它应该使用另一个时钟域以避免自我杀死（只有复位线应该不同） |

调试插件的内部实现方式可减少此插件的面积使用和FMax影响。

以下是访问它的简单总线，rsp在请求后一个周期到来：

```scala
case class DebugExtensionCmd() extends Bundle{
  val wr = Bool
  val address = UInt(8 bit)
  val data = Bits(32 bit)
}
case class DebugExtensionRsp() extends Bundle{
  val data = Bits(32 bit)
}

case class DebugExtensionBus() extends Bundle with IMasterSlave{
  val cmd = Stream(DebugExtensionCmd())
  val rsp = DebugExtensionRsp()

  override def asMaster(): Unit = {
    master(cmd)
    in(rsp)
  }
}
```


以下是寄存器映射：

```
读地址0x00 ->
  bit 0  : resetIt
  bit 1  : haltIt
  bit 2  : isPipBusy
  bit 3  : haltedByBreak
  bit 4  : stepIt
写地址0x00 ->
  bit 4  : stepIt
  bit 16 : 设置resetIt
  bit 17 : 设置haltIt
  bit 24 : 清除resetIt
  bit 25 : 清除haltIt和haltedByBreak

读地址0x04 ->
  bits (31 downto 0) : 最后写入寄存器文件的值
写地址0x04 ->
  bits (31 downto 0) : 应推入CPU流水线用于调试目的的指令
```

OpenOCD端口在：https://github.com/SpinalHDL/openocd_riscv

#### EmbeddedRiscvJtag

VexRiscv还支持官方RISC-V调试规范（感谢Efinix的资助！）。

要启用它，您需要将EmbeddedRiscvJtag添加到插件列表：

```scala
new EmbeddedRiscvJtag(
  p = DebugTransportModuleParameter(
    addressWidth = 7,
    version      = 1,
    idle         = 7
  ),
  withTunneling = false,
  withTap = true,
  jtagId = 0x10002FFF
)
```

并在CsrPlugin配置中打开withPrivilegedDebug选项。

以下是连接用的openocd tcl脚本示例：

```tcl
# 在这里添加您的JTAG适配器设置

set _CHIPNAME riscv
jtag newtap $_CHIPNAME cpu -irlen 5 -expected-id 0x10002FFF

set _TARGETNAME $_CHIPNAME.cpu

target create $_TARGETNAME.0 riscv -chain-position $_TARGETNAME

init
halt
```

完整示例在GenFullWithOfficialRiscvDebug.scala中

##### 隧道JTAG

EmbeddedRiscvJtag插件也可以与隧道JTAG一起使用。这允许使用用于配置FPGA的同一根电缆进行调试。

这使用FPGA特定的原语进行JTAG访问（例如Xilinx BSCANE2）：
```scala
val xilJtag = BSCANE2(userId = 4) // 必须userId = 4
val jtagClockDomain = ClockDomain(
  clock = xilJtag.TCK
)
```

然后，必须将EmbeddedRiscvJtag插件配置为隧道传输，不使用TAP。注意，调试时钟域必须与CPU时钟域分开复位。

```scala
// 在插件中
new EmbeddedRiscvJtag(
  p = DebugTransportModuleParameter(
    addressWidth = 7,
    version      = 1,
    idle         = 7
  ),
  withTunneling = true,
  withTap = false,
  debugCd = debugClockDomain,
  jtagCd = jtagClockDomain
)
```

然后将EmbeddedRiscvJtag连接到FPGA特定的JTAG原语：

```scala
for (plugin <- cpuConfig.plugins) plugin match {
  case plugin: EmbeddedRiscvJtag => {
    plugin.jtagInstruction <> xilJtag.toJtagTapInstructionCtrl()
  }
  case _ =>
}
```

以下是在Xilinx 7系列FPGA上连接的OpenOCD TCL脚本示例：

```tcl
# 在这里添加您的JTAG适配器设置

source [find cpld/xilinx-xc7.cfg]
set TAP_NAME xc7.tap

set _TARGETNAME cpu
target create $_TARGETNAME.0 riscv -chain-position $TAP_NAME
riscv use_bscan_tunnel 6 1

init
halt
```

#### YamlPlugin

此插件为其他插件提供服务，生成描述CPU配置的有用Yaml文件。例如，它包含刷新数据缓存所需的指令序列（openocd使用的信息）。

#### FpuPlugin

允许将内部或外部FPU集成到VexRiscv中（参见FPU章节）

| 参数 | 类型 | 描述 |
| ------ | ----------- | ------ |
| externalFpu   | Boolean | 当为false时，FPU在Vex中实例化，否则插件有一个`port`接口，您可以连接外部FPU |
| p   | FpuParameter | 连接FPU将使用的参数创建 |

#### AesPlugin

此插件允许通过使用内部ROM来解决SBOX和置换来加速AES加密/解密，在实践中允许在大约21个周期内执行一个AES轮次。

有关更多文档，请查看src/main/scala/vexriscv/plugin/AesPlugin.scala，软件C驱动程序可在此处找到：https://github.com/SpinalHDL/SaxonSoc/blob/dev-0.3/software/standalone/driver/aes_custom.h

它也被移植到libressl，补丁如下：
https://github.com/SpinalHDL/buildroot-spinal-saxon/blob/main/patches/libressl/0000-vexriscv-aes.patch

在linux中运行libressl时观察到4倍加速。https://github.com/SpinalHDL/SaxonSoc/pull/53#issuecomment-730133020