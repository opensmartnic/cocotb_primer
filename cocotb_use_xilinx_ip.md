### cocotb调用vivado中的IP核

有时候我们会希望用cocotb写TB文件来验证在vivado中的设计，也常涉及到vivado中的IP核。那么如何使cocotb中的代码可以用上这些IP核？

其实，这个问题需要再进一步转化。因为从cocotb的视角看，它并不清楚vivado的存在，它仅和仿真软件交互。那么vivado的IP核如何能被仿真软件所使用，就成了这个问题的等价问题。

首先是选择仿真软件，查阅*"Vivado Design Suite User Guide: Release Notes, Installation and Licensing"* (UG973) 或 https://support.xilinx.com/s/article/68324?language=en_US可以知道vivado支持的仿真软件（当然，还要求所选择的仿真软件被cocotb所支持，https://docs.cocotb.org/en/stable/simulator_support.html）

举个案例观察具体的过程。以modelsim为例

1、要在cocotb中使用modelsim仿真并不复杂，参考[这里](basic.md)；但要同时还使用vivado的IP核，则还需要事先进行编译，编译的方法可参考https://support.xilinx.com/s/article/64083?language=en_US, 编译完成后在指定的结果目录中有`modelsim.ini`，内记录各IP核所在路径，需要将这些内容和modelsim安装目录下的`modelsim.ini`文件合并。

2、在vivado中创建一个blockdesign，实例化一个BlockRam和它的AXI控制器，并将AXI协议转换为AXILite

![捕获](assets/ex_block_design_of_use_xilinx_ip.pgn)

3、在cocotb中编写代码实现写入和读出，具体python代码如下：

```python
import cocotb
from cocotb.triggers import Timer
from cocotb.clock import Clock
from cocotbext.axi import *

@cocotb.test()
async def my_first_test(dut):
    # 创建时钟，并等待时钟锁定
    await cocotb.start(Clock(dut.clk_in1_0, 10.0, 'ns').start())
    await Timer(3, "us")
    # 创建一个axilite总线的接口实例，用于读写
    axil_master = AxiLiteMaster(AxiLiteBus.from_prefix(dut, "S_AXI_0"), dut.clk_in1_0, dut.reset_0)
    # dut._log.info('hello')
    await axil_master.write(0x0000, b'1111')
    # 在这里可以观察到有数据写入到存储
    
    data = await axil_master.read(0x0000, 4)
    dut._log.info(data.data)
    # 在这里可以从日志中发现读取回来的数据，应等于写入的内容

    await Timer(1, "us")
```

4、在makefile中，添加源代码路径，需要将综合后生成的代码路径添加其中；在makefile中添加IP核所在的库文件名称；最后形成的makefile如下：

```makefile
# This file is public domain, it can be freely copied without restrictions.
# SPDX-License-Identifier: CC0-1.0

# Makefile

# defaults
SIM ?= modelsim
TOPLEVEL_LANG ?= verilog

# 在这里添加HDL文件，可以是verilog和vhd文件，视自己编写的文件类型和xilinx IP综合后产生的文件类型而定
VERILOG_SOURCES += /E/fpga/Vivado/2019.2/data/verilog/src/glbl.v
VERILOG_SOURCES += /E/Share/01MK7160-V2020/08_myprojs/cocotb_use_xilinx_ip/cocotb_use_xilinx_ip.srcs/sources_1/bd/design_1/synth/*.v
VERILOG_SOURCES += /E/Share/01MK7160-V2020/08_myprojs/cocotb_use_xilinx_ip/cocotb_use_xilinx_ip.srcs/sources_1/bd/design_1/ip/design_1_clk_wiz_0_0/design_1_clk_wiz_0_0.v
VERILOG_SOURCES += /E/Share/01MK7160-V2020/08_myprojs/cocotb_use_xilinx_ip/cocotb_use_xilinx_ip.srcs/sources_1/bd/design_1/ip/design_1_clk_wiz_0_0/design_1_clk_wiz_0_0_clk_wiz.v
VHDL_SOURCES += /E/Share/01MK7160-V2020/08_myprojs/cocotb_use_xilinx_ip/cocotb_use_xilinx_ip.srcs/sources_1/bd/design_1/ip/design_1_blk_mem_gen_0_0/synth/*.vhd
VHDL_SOURCES += /E/Share/01MK7160-V2020/08_myprojs/cocotb_use_xilinx_ip/cocotb_use_xilinx_ip.srcs/sources_1/bd/design_1/ip/design_1_axi_bram_ctrl_0_0/synth/*.vhd
VERILOG_SOURCES += /E/Share/01MK7160-V2020/08_myprojs/cocotb_use_xilinx_ip/cocotb_use_xilinx_ip.srcs/sources_1/bd/design_1/ip/design_1_axi_protocol_convert_0_0/synth/*.v
VERILOG_SOURCES += /E/Share/01MK7160-V2020/08_myprojs/cocotb_use_xilinx_ip/cocotb_use_xilinx_ip.srcs/sources_1/bd/design_1/ipshared/c4a6/hdl/*.v

# TOPLEVEL is the name of the toplevel module in your Verilog or VHDL file
TOPLEVEL = design_1

# MODULE is the basename of the Python test file
MODULE = test_xilinx_ip

# 在这里添加modelsim所需要的库名称
SIM_ARGS = -L blk_mem_gen_v8_4_4 -L axi_bram_ctrl_v4_1_2 -L unisims_ver -L unimacro_ver -L secureip -L xpm -L smartconnect_v1_0 -L xlconstant_v1_1_6 work.glbl 

# GUI=1

# include cocotb's make rules to take care of the simulator setup
include $(shell cocotb-config --makefiles)/Makefile.sim

```

5、make执行

部分cocotb的输出如下：

```
... 省略之前的部分
  3000.00ns INFO     cocotb.design_1.S_AXI_0              rdata width: 32 bits
  3000.00ns INFO     cocotb.design_1.S_AXI_0              rready width: 1 bits
  3000.00ns INFO     cocotb.design_1.S_AXI_0              rresp width: 2 bits
  3000.00ns INFO     cocotb.design_1.S_AXI_0              rvalid width: 1 bits
  3000.00ns INFO     cocotb.design_1.S_AXI_0            Reset de-asserted
  3000.00ns INFO     cocotb.design_1.S_AXI_0            Write start addr: 0x00000000 prot: AxiProt.NONSECURE data: 31 31 31 31
  3020.00ns INFO     cocotb.design_1.S_AXI_0            Write complete addr: 0x00000000 prot: AxiProt.NONSECURE resp: AxiResp.OKAY length: 4
  3020.00ns INFO     cocotb.design_1.S_AXI_0            Read start addr: 0x00000000 prot: AxiProt.NONSECURE length: 4
  3070.00ns INFO     cocotb.design_1.S_AXI_0            Read complete addr: 0x00000000 prot: AxiProt.NONSECURE resp: AxiResp.OKAY data: 31 31 31 31
  3070.00ns INFO     cocotb.design_1                    b'1111'
  4070.00ns INFO     cocotb.regression                  my_first_test passed
  4070.00ns INFO     cocotb.regression                  **************************************************************************************
                                                        ** TEST                          STATUS  SIM TIME (ns)  REAL TIME (s)  RATIO (ns/s) **
                                                        **************************************************************************************
                                                        ** test_xilinx_ip.my_first_test   PASS        4070.00           0.09      44431.78  **
                                                        **************************************************************************************
                                                        ** TESTS=1 PASS=1 FAIL=0 SKIP=0               4070.00           0.42       9793.13  **
                                                        **************************************************************************************
```

至此，使用cocotb仿真vivadoIP核完成。

说明：在第3步中，我们使用了一个额外的库(cocotbext-axi)来实现对AXI总线的读写，具体将在下一篇中介绍。



