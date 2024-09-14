# Qemu 介绍

xv6的操作系统就运行在Qemu上面，所以很有必要先对它有一个大概的了解，更方便我们去使用



## 运行：

### qemu -kernel

- qemu -kernel 是否真的可初始化的时候直接加载到物理内存中？

当使用 `qemu -kernel` 命令时，QEMU 会加载指定的内核映像文件并启动一个虚拟机实例来运行该内核。这个过程大致可以描述如下：

1. **加载内核映像文件**: QEMU 会加载 `-kernel` 参数指定的内核映像文件到虚拟机的内存中。
2. **设置启动参数**: QEMU 可能会根据命令行参数或默认设置为内核指定一些启动参数，比如内存大小、CPU 类型等。
3. **初始化虚拟硬件**: 在加载内核前，QEMU 会初始化虚拟硬件环境，包括模拟处理器、设备、内存等。
4. **跳转到内核入口**: 加载完内核映像后，QEMU 会设置处理器的程序计数器（PC）指向内核的入口地址，并开始执行内核代码。
5. **内核启动**: 内核开始在虚拟硬件环境中初始化系统，包括设置页表、初始化设备、加载驱动程序等。

在使用 `qemu -kernel` 命令加载内核时，默认情况下，QEMU 会从内核映像文件中指定的入口地址开始执行内核代码。通常情况下，内核映像文件会在编译时指定一个特定的入口地址，该地址标志着内核代码的起始位置。

## 退出：

To quit qemu type: **Ctrl-a x** (press **Ctrl** and **a** at the same time, followed by **x**).



## 启动：

###  1. 常规启动参数

```Makefile
QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```



> 以下是 QEMU 模拟器启动参数的具体含义：
>
> 1. `-machine virt`: 指定使用 "virt" 机器类型，这是一种通用的虚拟机配置，适用于多种操作系统和应用程序。
> 2. `-bios none`: 禁用 BIOS，这意味着模拟器将不会加载任何 BIOS。在此配置中，模拟器直接从内核启动。
> 3. `-kernel $K/kernel`: 指定内核文件的路径。这里的 `$K/kernel` 是一个变量，表示内核文件的路径。
> 4. `-m 128M`: 分配给模拟器的内存大小为 128 MB。
> 5. `-smp $(CPUS)`: 指定模拟器中的处理器数量。这里的 `$(CPUS)` 是一个变量，表示处理器的数量。
> 6. `-nographic`: 禁用图形化界面，以纯文本方式运行模拟器。
> 7. `-global virtio-mmio.force-legacy=false`: 设置 virtio-mmio 总线为非遗留模式。virtio-mmio 是一种虚拟设备总线，用于连接虚拟机和宿主机之间的块设备和其他设备。
> 8. `-drive file=fs.img,if=none,format=raw,id=x0`: 添加一个虚拟硬盘驱动器，`fs.img` 是硬盘镜像文件的路径，`if=none` 表示不将其连接到默认接口，`format=raw` 指定硬盘镜像的格式为原始二进制，`id=x0` 给驱动器分配一个唯一的标识符。
> 9. `-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0`: 添加一个 virtio 块设备，将前面创建的驱动器连接到 virtio-mmio 总线上。`drive=x0` 表示将驱动器 `x0` 连接到该设备上，`bus=virtio-mmio-bus.0` 指定设备连接到 virtio-mmio 总线的第一个设备槽上。
>
> 这些启动参数用于配置 QEMU 模拟器的运行环境，包括内核、内存、处理器、虚拟硬盘等方面。根据具体的需求，可以根据这些参数进行修改和调整。



### 2. 一个实际的网络驱动启动参数：

**在net的lab中有涉及到**

```C
-netdev user,id=net0,hostfwd=udp::26999-:2000 
-object filter-dump,id=net0,netdev=net0,file=packets.pcap 
-device e1000,netdev=net0,bus=pcie.0
```

1. `-netdev user,id=net0,hostfwd=udp::26999-:2000`
   1. `-netdev user`: 配置一个用户模式网络后端。用户模式网络提供简单的 NAT 路由，适用于无需高级网络功能的场景。
   2. `id=net0`: 为该网络设备分配一个 ID，便于后续引用。
   3. `hostfwd=udp::26999-:2000`: 设置主机到虚拟机的 UDP 端口转发规则。这条规则将主机的 UDP 端口 `26999` 转发到虚拟机的 UDP 端口 `2000`。
   4.  To forward host ports to your guest, use **-netdev user,id=n0,hostfwd=hostip:hostport-guestip:guestport**
   5.  具体格式为 `hostfwd=udp::主机端口-虚拟机IP:虚拟机端口`。这里的 `虚拟机IP` 省略，默认转发到虚拟机的相应端口。
2. `-object filter-dump,id=net0,netdev=net0,file=packets.pcap`
   1. `-object filter-dump`: 配置一个数据包捕获过滤器对象。
   2. `id=net0`: 为该对象分配一个 ID。
   3. `netdev=net0`: 指定与 `-netdev` 中配置的网络设备关联。
   4. `file=packets.pcap`: 指定捕获的数据包输出到 `packets.pcap` 文件中。这个文件将包含通过 `net0` 网络设备的所有数据包。
3. `-device e1000,netdev=net0,bus=pcie.0`
   1. `-device e1000`: 启动一个虚拟网络设备，类型为 `e1000`（英特尔的 1Gbps 网络接口卡）。
   2. `netdev=net0`: 将该设备绑定到先前定义的 `net0` 网络设备。
   3. `bus=pcie.0`: 指定该网络设备连接到的总线类型和位置。这里是 `pcie.0`，表示连接到 PCI Express 总线。



### 3. Qemu 对网络驱动的支持：

#### 虚拟化网络设备

QEMU模拟多种网络设备，这些设备可以通过命令行选项（如`-device`）添加到虚拟机配置中。一些常见的模拟网络设备包括：

- e1000：Intel的一个常见网络接口卡模型，广泛支持于多种操作系统。
- virtio-net：一个高性能的网络设备模型，专为虚拟化环境设计。使用virtio-net需要操作系统支持virtio驱动。

#### 驱动程序接口

- virtio：Virtio是一个标准化的接口，专为虚拟化环境中的设备（包括网络设备）设计。操作系统需要实现virtio协议来与基于virtio的虚拟设备（如virtio-net）进行通信。Virtio定义了一组设备发现、配置和数据传输的机制，这使得它比传统的设备模拟更为高效。
- 标准设备接口：对于非virtio的设备模拟（如e1000），QEMU提供了与物理设备类似的接口。你的操作系统需要包含或实现相应设备的驱动程序，这些驱动程序需要能够识别设备、配置设备，并通过设备进行数据的发送和接收。

总的来说，QEMU提供了通过模拟网络设备与虚拟机中的操作系统进行交互的能力，但没有实现网络通信功能



## shell xv6 调试方法：

To use gdb with xv6, run make **make qemu-gdb** in one window, run **gdb-multiarch** (or **riscv64-linux-gnu-gdb**) 

```C
make qumu-gdb

另外一个窗口执行：
gdb-multiarch

然后就可以执行 break 打断点，以及 c 来开始调试
```

### 关于gdb种类的介绍：

`gdb-multiarch` 和 `riscv64-linux-gnu-gdb` 是GNU Debugger (GDB) 的两种不同变体，它们都是用于调试程序，但主要区别在于它们的适用范围和目标体系结构。

#### gdb-multiarch (调试时主要使用)

- 适用范围：`gdb-multiarch` 是一种支持多种体系结构的GDB版本。这意味着它包括了对多个不同处理器体系结构的支持，可以用单一的GDB实例来调试多种不同体系结构的程序。这种版本的GDB特别适合于开发者和测试人员，他们需要处理多种硬件平台上的软件。
- 灵活性：由于其多体系结构支持特性，`gdb-multiarch` 提供了高度的灵活性，允许用户在同一个调试器中切换不同的体系结构目标。
- 安装和使用：通常在支持多种体系结构的开发环境中安装和使用，如在使用交叉编译工具链的环境中。

#### riscv64-linux-gnu-gdb

- 专用性：`riscv64-linux-gnu-gdb` 是专为RISC-V 64位体系结构（具体是Linux目标）定制的GDB版本。这种版本的GDB预配置了对RISC-V体系结构的支持，优化了调试这一体系结构程序的功能。
- 体系结构特定的特性：该版本可能包括针对RISC-V体系结构的特殊优化或特性，例如更好的寄存器显示、特殊处理器指令的支持等。
- 使用场景：主要用于开发和调试在RISC-V 64位平台上运行的应用程序，特别是那些在Linux操作系统上运行的应用程序。



