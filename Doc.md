# <center> lowRISC 64-bit SOC </center>

## 1. RTL Level One

### 1.1 Top: chip\_top
#### clk\_gen (时钟分频器)
>input: clk\_in1:100MHz
>
>output: clk\_out1: 200MHz	--> mig\_sys\_clk  / 接入DRAM Ctrl
>
>output: clk\_out2: 60MHz	--> clk\_io\_uart / for debug 初始SOC中未接入任何模块
>
>output: clk\_out3: 120MHz	--> clk\_pixel / 接入periph_soc内fstore2， 用于VGA的输出
>
>output: clk\_out4: 50MHz	--> clk\_rmii\_quad
>
>output: clk\_out5: 50MHz	--> clk\_rmii / 接入periph\_soc 内的framing\_top，RMII为MAC层和PHY层之间的传输借口
>

#### dram_ctl (DDR控制器)

> 内接 Rocket， 外输出DDR

#### BramCtl（Human interface and miscellaneous devices）

>Data With: 64
>Data Depth: 32768
>Protocol: AXI
>
>处理MMIO：mmio\_master\_nasti
>处理HID：指交互的信号，接 periph\_soc
>

### periph_soc

> VGA
> Eth
> Uart
> Phy
> LED
> SD 卡
> 开关（i_dip）
> 
> 受 BramCtrl 控制

### rocket
> L2 系列：全部接地
> 
> mem\_axi4 输入/输出：来自/接入 dram\_ctl
> 
> mmio 输入/输出：来自/接入 BramCtrl

## 2. RTL Level Two

### periph_soc

periph_soc 从所有外设读取来的数据，并不走RAM过，而是直接传递给Rocket做处理

#### keyborad+mouse+VGA
- keyb\_mouse/keyb\_fifo: 用于处理PS2
- the\_fstore: 用于VGA的图像输出，内涵一个双端口RAM和一个文字转换模块，将一个ASCII转成相应大小的图像

#### uart
- uart\_rx\_fifo/uart\_tx\_fifo: 收发fifo
- i\_uart: uart 状态机
- rx\_delay: 延迟模块

#### SD卡
- sd\_top: 读写口直接接输入/输出，SD的读写不走存储器，而是直接从periph_soc给一根终端信号到rocket，直接从SD卡读数据

#### 以太网
- framing\_top
由于物理层的处理已经集成在开发板上，所以这里FPGA的接口实际上是MAC和PHY层之间的接口信号，具体信号的意义可见[以太网信号解析](https://blog.csdn.net/zcshoucsdn/article/details/80090802#comments)。


## 3. 地址映射

|  Usage | actual address spaces   | mapped address spaces | Type |
|  ----  | ----  | ----  | ----  |
|Memory section |	(0x00000000 - 0x7FFFFFFF) | (0x00000000 - 0x7FFFFFFF) | Mem | 
|On-chip BRAM (64 KB)	|(0x00000000 - 0x0000FFFF)|(0x00000000 - 0x0000FFFF) | Mem | 
|DDR DRAM | 	(0x40000000 - 0x7FFFFFFF) | (0x40000000 - 0x7FFFFFFF) | Mem | 
|I/O section |	(0x80000000 - 0x8FFFFFFF) | (0x80000000 - 0x8FFFFFFF) | I/O | 
|UART & SD|(0x80000000 - 0x8001FFFF)|(0x80000000 - 0x8001FFFF)|I/O|

## 4. 上电过程

本实验的的SOC文件存储在 lowrisc-chip-refresh-v0.6/src/main/verilog 中，Rocket-chip的Chisel代码为现场编译。

首先加载 boot.mem，给出Uart的提示信息，这段代码可以从Make过程中的加载指令找到，该指令实现和编译目标工程的链接：
> ln -s /home/macbookpro/RISCV/lowrisc-chip-refresh-v0.6/fpga/board/nexys4\_ddr/src/boot.mem lowrisc-chip-imp/lowrisc-chip-imp.runs/synth\_1/boot.mem

在chip_top.sv中，创建bram的时候读取相应的文件，用于初始化，即第一段boot程序是烧写在SOC内部的。
>
>// BRAM controller
>
>logic ram_clk, ram_rst, ram_en;
>
>logic [7:0] ram_we;
>
>/* Detials Omitted (Around Line 600) */
>
>initial $readmemh("boot.mem", ram);
>

上电后从BRAM加载这一段代码完成初始化。在demo里，SPI Flash的端口约束被注释，因此make cfgmem实际上就是无效的。

下属指令用于完成boot.bin的传输
>../../common/script/recvRawEth -r -s $IP boot.bin

之后，以太网传输boot.bin，通过 periph\_soc 传递给rocket进行处理。
		











