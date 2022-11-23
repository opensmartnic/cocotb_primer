### cocotb概念和使用方法

本文主要是既有文档的翻译和整理：https://docs.cocotb.org/

#### 什么是cocotb？

cocotb是一个可以用python编写[testbench文件](https://en.wikipedia.org/wiki/Test_bench)的工具，使用它可以方便地生成输入仿真所需的激励信号并用python代码分析和校验仿真的输出。

参考：https://docs.cocotb.org/en/stable/

#### 安装

有3个部分需要先准备就绪：

* python3.6+
* GNU Make3+，负责配置和引导启动仿真
* HDL仿真器，负责实际执行仿真过程

1、python环境上推荐使用conda进行安装。

2、make环境如果是linux，则使用软件包安装，如`apt install make`；如果是windows，则也推荐依然通过conda安装，如

```
pip install m2-base m2-make
```

3、[HDL](https://en.wikipedia.org/wiki/Hardware_description_language)仿真器看个人需要安装一种或多种。cocotb本身支持多种仿真器，见[这里](https://docs.cocotb.org/en/stable/simulator_support.html#simulator-support)。如，笔者使用的Modelsim和Icarus都在其中。注意，要求仿真器是可以通过命令行直接调用（即正确配置了PATH环境变量），如笔者配置的windows环境中，可以在powershell中直接使用`vsim.exe`来启动Modelsim仿真界面。

4、完成上述3个软件安装后，可以在python安装包的环境中执行下列命令：

```
pip install cocotb
```

安装完成的标志是，你可以在命令行中执行cocotb-config[.exe]来查看配置，以及可以在python文件中正常`import cocotb`：

```
(base) PS E:\xx\projs\test_gen> cocotb-config.exe
usage: cocotb-config [-h] [--prefix] [--share] [--makefiles] [--python-bin] [--help-vars] [--libpython] [--lib-dir] [--lib-name INTERFACE SIMULATOR] [--lib-name-path INTERFACE SIMULATOR] [-v]

optional arguments:
  -h, --help            show this help message and exit
  --prefix              echo the package-prefix of cocotb
...
```

参考：https://docs.cocotb.org/en/stable/install.html

#### 初识

要启动一次仿真测试，需要有3个部分的代码：

* verilog和vhdl描述的硬件电路，这也是仿真的对象
* 用cocotb编写的testbench
* 放置在makefile中的配置

1、示例：创建一个system verilog文件：

```systemverilog
// save as my_design.sv
module my_design(input logic clk);

  timeunit 1s;
  timeprecision 1ns;

  logic my_signal_1;
  logic my_signal_2;

  assign my_signal_1 = 1'bx;
  assign my_signal_2 = 0;

endmodule
```

2、编写tb文件

```python
# save as test_my_design.py

import cocotb
from cocotb.triggers import Timer

@cocotb.test()
async def my_first_test(dut):
    """Try accessing the design."""

    for cycle in range(10):
        dut.clk.value = 0
        await Timer(1, units="ns")
        dut.clk.value = 1
        await Timer(1, units="ns")

    dut._log.info("my_signal_1 is %s", dut.my_signal_1.value)
    assert dut.my_signal_2.value[0] == 0, "my_signal_2[0] is not 0!"
```

和一般TB文件用HDL语言编写不同，这里完全用python编写。在这个TB文件里，产生了10个时钟（频率为500MHz），之后读取`my_signal_1`的值，并校验`my_signal_2==0`是否成立。

（1）使用`cocotb.test()`对测试函数进行注解

（2）测试函数参数`dut`会由上层调用传入，代表仿真对象DUT（Design Under Test），可以获取DUT信号、端口、参数等，使用`.value`来获取和设置具体的值

（3）测试函数需要添加async关键字，这将使得函数的返回是一个[协程(coroutine)](https://docs.python.org/3/library/asyncio-task.html#coroutine)，并进而被cocotb的控制程序所启动

3、编写Makefile文件

```makefile
# save as Makefile

# defaults
SIM ?= modelsim
TOPLEVEL_LANG ?= verilog

VERILOG_SOURCES += $(PWD)/my_design.sv
# use VHDL_SOURCES for VHDL files

# TOPLEVEL is the name of the toplevel module in your Verilog or VHDL file
TOPLEVEL = my_design

# MODULE is the basename of the Python test file
MODULE = test_my_design
# 以上是第一部分

# 以下为第二部分
# include cocotb's make rules to take care of the simulator setup
include $(shell cocotb-config --makefiles)/Makefile.sim
```

在Makefile中，第一部分负责配置具体的选项，包括：

* 使用的仿真器软件，由`SIM`变量指定。注：示例中的`?=`是Makefile的语法，表示没有定义时则使用右值，如，从命令启动时仍然可以指定该变量的值，如`make SIM=icarus`是会比`?=`指定的值优先生效。
* 顶层模块使用的编程语言，支持verilog和vhdl两种，由`TOPLEVEL_LANG`变量指定。这里只是指顶层模块的编程语言，可以支持混合语言
* HDL文件列表，verilog和vhdl分别的使用`VERILOG_SOURCES`和`VHDL_SOURCES`串联编写
* 顶层模块的名称，由`TOPLEVEL`指定，即DUT的入口。注：有些仿真器还要求顶层模块所在文件的具有相同的文件名
* 用python编写的TB文件的名称（不带后缀），由`MODULE`变量指定

在Makefile中的第二部分，则是实际启动仿真相关的规则，cocotb已经编写了模板，直接引用即可，无需改动。

4、将上述文件放置在同一目录下，并执行：

```bash
make
```

将启动仿真，打印输出，类似如下：

```
...
# ** Note: $finish
#    Time: 20010 ps  Iteration: 0  Instance: /my_design
# End time: 16:05:12 on Nov 18,2022, Elapsed time: 0:00:01
# Errors: 0, Warnings: 1
make[1]: Leaving directory '/e/xx/projs/cocotb_quickstart'
```

同时TB文件也会产生日志信息，类似如下：

```
     0.00ns INFO     cocotb                             Running on ModelSim SE-64 version 2019.2 2019.04
     0.00ns INFO     cocotb                             Running tests with cocotb v1.7.0.dev0 from C:\ProgramData\Anaconda3\lib\site-packages\cocotb
     0.00ns INFO     cocotb                             Seeding Python random module with 1668758855
     0.00ns INFO     cocotb.regression                  Found test test_my_design.my_first_test
     0.00ns INFO     cocotb.regression                  running my_first_test (1/1)
                                                          Try accessing the design.
    20.00ns INFO     cocotb.my_design                   my_signal_1 is x
    21.00ns INFO     cocotb.regression                  my_first_test passed
    21.00ns INFO     cocotb.regression                  **************************************************************************************
                                                        ** TEST                          STATUS  SIM TIME (ns)  REAL TIME (s)  RATIO (ns/s) **
                                                        **************************************************************************************
                                                        ** test_my_design.my_first_test   PASS          21.00           0.00      10492.01  **
                                                        **************************************************************************************
                                                        ** TESTS=1 PASS=1 FAIL=0 SKIP=0                 21.00           0.30         69.54  **
                                                        **************************************************************************************
```

可以看到`dut._log.info(...)`信息有对应的显示。

至此完成了一次仿真，输入的时钟由python代码控制生成，并在python中可以读取信号值。

参考：https://docs.cocotb.org/en/stable/quickstart.html

#### 背后发生了什么？

以windows下使用modelsim仿真为例：

```
make
└── vsim
    └── vlog.exe
    └── vsimk.exe
        └── cocotbvpi_modelsim.dll
            └── libpython.dll
```

运行`make`后将生成调用modelsim所需的命令行，类似这样：

```
/c/modeltech64_2019.2/win64/vsim -c -64  -do sim_build/runsim.do  2>&1 | tee sim_build/sim.log
```

而后`vsim`调用`vlog.exe`对`hdl文件`进行编译，然后调用`vsimk.exe`启动仿真。其中`vsimk.exe`会调用`cocotbvpi_modelsim.dll`，并进而调用`libpython.dll`来实际执行用python编写的`TB文件`。

#### 更多示例

1、并发控制

实际使用中可能需要多个函数/协程来分别处理输出信号和生成新的输入，如将时钟信号的生成单独分离，这可以通过生成一个新的协程[的任务]来实现，如：

```python
# test_my_design.py (extended)

import cocotb
from cocotb.triggers import FallingEdge, Timer

async def generate_clock(dut):
    """Generate clock pulses."""

    for cycle in range(10):
        dut.clk.value = 0
        await Timer(1, units="ns")
        dut.clk.value = 1
        await Timer(1, units="ns")

@cocotb.test()
async def my_second_test(dut):
    """Try accessing the design."""
    cocotb.start_soon(generate_clock(dut))  # 在这里启动新的协程

    await Timer(5, units="ns")  # wait a bit
    await FallingEdge(dut.clk)  # wait for falling edge/"negedge"

    dut._log.info("my_signal_1 is %s", dut.my_signal_1.value)
    assert dut.my_signal_2.value[0] == 0, "my_signal_2[0] is not 0!"
```

注意，当`cocotb.test()`注解的函数执行完成后，不管其生成的子协程是否运行完成，测试过程都将完成并退出。

多协程并发，看起来是会引起一些时间线先后而导致的结果不同，如协程1马上发起对信号1的上升沿等待，而协程2则需要更多的处理时间后，才发起对信号1的上升沿等待，这看起来是会导致协程1事件先被响应。更或者，两个协程需要的处理有随机性，那是否会导致多次仿真的结果不具有确定性？简短地回答这个问题是：不会，cocotb有同步机制使得多协程统一到达等待仿真器响应，处理时间的长短丝毫不影响仿真的前后顺序。更具体地，将在[cocotb中的时间线](timeline.md)中讨论。

2、多个测试函数

可以在tb文件中编写多个被`cocotb.test()`注解的测试函数，他们将按照定义的顺序被**依次**调用。如

```python
@cocotb.test()
async def test3(dut):
    raise cocotb.result.TestFailure("test3")
    # dut._log.info("test3")

@cocotb.test()
async def test2(dut):
    dut._log.info("test2")
```

将生成类似的输出：

```log
     0.00ns INFO     cocotb.regression                  running test3 (1/2)
     0.01ns INFO     cocotb.regression                  test3 failed
                                                        Traceback (most recent call last):
                                                          File "E:\xx\projs\cocotb_quickstart\test_my_design.py", line 49, in test3
                                                            raise cocotb.result.TestFailure("test3")
                                                        cocotb.result.TestFailure: test3
     0.01ns INFO     cocotb.regression                  running test2 (2/2)
     0.01ns INFO     cocotb.my_design                   test2
     0.02ns INFO     cocotb.regression                  test2 passed
```

