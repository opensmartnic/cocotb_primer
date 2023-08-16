### cocotbext.axi项目介绍

 **A**dvanced e**X**tensible **I**nterface（AXI接口，Arm发起的总线接口标准，https://support.xilinx.com/s/article/1053914?language=en_US）可以说是xilinx IP核之间最为常见的交互协议，大量IP之间交换数据、发送控制信号都通过它进行。具体地，AXI又分为AXI Full（以下也简称为AXI，可能有点混淆...），AXIlite、AXIStream三种类型。

当对某个模块进行仿真时，若它采用AXI标准作为接口，那么则需要编写一套流程控制代码来完成协议之间的匹配，这是有点复杂的，尤其是还涉及异步等待时。cocotbext.axi的出现，可以节省我们大量的时间。

题外话：这里要强烈推荐下alexforencich名下的项目，既有基础性的verilog项目如pcie、axi，也有相应的cocotb仿真项目可便捷地调试，同时他还是corundum（开源100G网卡）的创建人。总之，是个宝藏up主。

#### 一、cocotbext.axi的基本用法

模块的安装参考https://github.com/alexforencich/cocotbext-axi

AXI是master-slave架构，如果你的仿真对象是AXI的slave端，那么应该在python端代码创建一个master对象，然后在这个对象上进行发送或接收。创建的方式如下：

```python
from cocotbext.axi import AxiBus, AxiMaster

axi_master = AxiMaster(AxiBus.from_prefix(dut, "s_axi"), dut.clk, dut.rst)
```

初始化的参数要求有AxiBus、时钟和复位信号。

这里的AxiBus提供了AXI接口的信号线接口，使用仿真对象和信号线的前缀(“s_axi")进行初始化，如上面的例子，就会自动关联dut对象的s_axi_awaddr, s_axi_awvalid, s_axi_wdata等等一系列信号。AxiBus继承自cocotb本身提供的Bus基本类，并额外声明了需要哪些信号线（根据不同的AXI协议类型）。Bus本身还提供了两个重要的方法：drive和sample，分别用于对这些信号进行赋值和取值。

创建完master对象后就可以调用其read、write进行读、写操作，需要使用await来阻塞调用（即命令结束就意味着已经成功写入或读出，否则将被阻塞）。

```python
await axi_master.write(0x0000, b'test')
data = await axi_master.read(0x0000, 4)
```

还有一种非阻塞的调用方式：

```python
write_op = axi_master.init_write(0x0000, b'test')
await write_op.wait()
resp = write_op.data
read_op = axi_master.init_read(0x0000, 4)
await read_op.wait()
resp = read_op.data
```

其中init_write返回的是一个[Event](https://docs.cocotb.org/en/stable/triggers.html#cocotb.triggers.Event)，这个Event会在写入完成后被触发，所以可以通过在其上的await的来实现写入完成时机的同步。这种分两行代码的编程方式略显复杂，但同时也提供了一种灵活性，比如同时发起多个读写操作（因为还没触发await，所以对仿真器而言，这些操作是同时发起的），并且还可以利用cocotb提供的多事件组成监听方式，如Combine、First、with_timeout等，来进行更多的细节操控。

以上是AxiMaster提供的主要功能，AxiLiteMaster的接口也一样，尽管对AxiLite而言，一次只能读写一个字，但这个细节被隐藏了。

如果你的仿真对象是master端，那么应该在python端创建一个Slave对象。方式是类似的：

```python
from cocotbext.axi import AxiBus, AxiSlave, MemoryRegion

axi_slave = AxiSlave(AxiBus.from_prefix(dut, "m_axi"), dut.clk, dut.rst)
region = MemoryRegion(2**axi_slave.read_if.address_width)
axi_slave.target = region
```

额外有些不同的是slave作为被动响应端，需要有一个池子接收和存储master发送的数据，并能在master发起读请求时，将数据送出。作为一个slave设备，其接收数据也是在接收控制，处理完毕后给出相应的回应。上面的例子相当于一个没有额外处理能力的slave设备，给什么存什么。如果你需要更复杂的slave功能，则需要自行编写。

对于这种纯接收的操作，还有更简洁的使用模式是直接定义一个AxiRam对象，它作为slave设备与master连接：

```python
from cocotbext.axi import AxiBus, AxiRam

axi_ram = AxiRam(AxiBus.from_prefix(dut, "m_axi"), dut.clk, dut.rst, size=2**32)
```

AxiRam还有其他的接口，如：write、read。注意不要误解以为slave端也可以像master那样发起读写，其实这里的write和read是对主机端的内存进行操作，所以也可以看到他们不需要使用阻塞调用。

```python
axi_ram.write(0x0000, b'test')
data = axi_ram.read(0x0000, 4)
axi_ram.hexdump(0x0000, 4, prefix="RAM")
```

上面的这些内容适用于AXI和AXILite，对于AXIStream，略有不同。AXIStream虽然仍有read/write接口，但更常使用的send/recv这样的操作。同样的，他们也需要阻塞调用。send（或write）上的阻塞调用，并不是等到发送完毕才返回，而是开始发送即返回，这是和AXI/AXILite的不同。如果要等待发送完成，则需要调用wait()方法：

```python
await axis_source.send(b'test data')
# wait for operation to complete (optional)
await axis_source.wait()
```

#### 二、cocotbext.axi的实现逻辑

作为开源项目，cocotbext.axi（以下简称项目）怎么实现的都在源码里，这里介绍我从代码中观察到的设计模式。

在这之前，先介绍cocotb提供的两个同步工具：[Event](https://docs.cocotb.org/en/stable/triggers.html#cocotb.triggers.Event)和[Queue](https://docs.cocotb.org/en/stable/library_reference.html#module-cocotb.queue)。

Event（事件）是cocotb的协程间进行同步的主要手段，其基本用法是在A协程中等待某事件，当B协程在某个时机设置该事件后，A协程被唤醒，从而实现协程间的同步。

Queue则是一个生产者-消费者模型，不同的协程可以分角色进行，如A负责把数据放进队列，而B则是常驻运行，当发现有数据时负责处理，没数据时则空闲等待。

这样的模式，在项目中反复出现。以AXI接口的发送或接收逻辑为例。

我们知道AXI协议中，发送端发起valid信号表示数据准备完毕完成，接收端发起ready信号表示可以接收，两者都OK时在时钟的上升沿进行了数据传输。这一逻辑，项目使用了一个常驻协程了完成，代码片段如下：

```python
    # 以下截取自：https://github.com/alexforencich/cocotbext-axi/blob/f2bf8c0ed8813974fc1dfef7b835233468ab6b90/cocotbext/axi/stream.py#L251C1-L279C55
    # 它负责master端数据的发送
    async def _run(self):
        has_valid = self.valid is not None
        has_ready = self.ready is not None

        clock_edge_event = RisingEdge(self.clock)

        while True:
            await clock_edge_event

            # read handshake signals
            ready_sample = not has_ready or self.ready.value
            valid_sample = not has_valid or self.valid.value

            if (ready_sample and valid_sample) or (not valid_sample):
                if not self.queue.empty() and not self.pause:
                    self.bus.drive(self.queue.get_nowait())
                    self.dequeue_event.set()
                    if has_valid:
                        self.valid.value = 1
                    self.active = True
                else:
                    if has_valid:
                        self.valid.value = 0
                    self.active = not self.queue.empty()
                    if self.queue.empty():
                        self.idle_event.set()
                        self.active_event.clear()

                        await self.active_event.wait()
```

作为一个常驻协程，会在每个时钟的上升沿处理一次数据，从队列里读取，并发送到总线上（或对端的信号线上）。至于数据怎么进队列的，则由更上一层的逻辑来负责。这里的上一层逻辑指AxiWriteCmd/AxiReadCmd这样的操作，在Cmd的这层抽象中，同样也有类似的模式，即“消费者-生成者”队列，有一个常驻协程负责监听这个队列，当有Cmd需要处理时，这个协程负责执行。另一个协程则负责将Cmd发送到队列中，并等待特定的事件完成。

为了便于表述，这里引入这些记号：

* AWQUEUE：写操作地址写入动作的队列，负责常驻处理该队列操作的协程称为coco_aw
* WQUEUE：写操作数据写入动作的队列，负责常驻处理该队列操作的协程称为coco_w
* WRQUEUE：写操作resp信号读动作的队列，负责常驻处理该队列操作的协程称为coco_wr
* WCMDQUEUE：写入命令的队列， 负责常驻处理该队列操作的协程称为coco_wcmd
* WRCMDQUEUE：读取写操作resp信号命令的队列，负责常驻处理该队列操作的协程称为coco_wrcmd

以AxiLite协议为例。当使用`await write(...)`这样的操作时，其背后是创建一个事件E，并将Cmd放入到WCMDQUEUE，然后就开始等待事件E完成。coco_wcmd发现有新的cmd进来后，开始启动写操作地址写入、写操作数据写入两个过程（也就是分别把地址和数据扔到AWQUEUE和WQUEUE中，然后由coco_aw和coco_w再自行分别去处理），同时还创建一个读取写做操的resp信号的命令，并放到到WRCMDQUEUE中。coco_wrcmd发现有需要处理的Cmd，就启动执行，并在执行完成后最终负责把事件E触发。由此完成一次写入过程。

我们之前说AXIStream中的send（或write）并不是等待发送完成，其实AXI、AXIlite中的写操作在执行时也不是等待完成，而是指放进了队列（也相当于启动了写），具体什么时候完成，是通过异步等待resp信号的到来来实现了。当然这是AXI、AXILite有resp信号而带来的便利。对于AXIStream，由于是流式报文，具体的完成动作则要通过捕捉tlast信号来实现。





