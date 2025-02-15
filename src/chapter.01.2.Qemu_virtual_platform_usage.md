# 1.2 如何使用Qemu平台调试

Qemu 是一个开源的虚拟机平台，可以提供多种虚拟硬件，可以让性能接近本机，常用于操作系统开发，嵌入式系统开发。

xv6的操作系统就运行在Qemu上面，所以很有必要先对它有一个大概的了解，更方便我们去使用



## 1. 运行：

To use gdb with xv6, run make **make qemu-gdb** in one window, run **gdb-multiarch** (or **riscv64-linux-gnu-gdb**) 

```C
第一个窗口运行：
make qumu-gdb

另外一个窗口执行：
gdb-multiarch
接着输入`c`来开始启动程序, 也可以通过执行 break 先打断点，再启动程序
```

这里不需要直接使用 qumu 的命令来启动，因为启动过程都封装在 makefile 文件中，所以直接使用make 命令来代为运行



## 2. 退出：

To quit qemu type: **Ctrl-a x** (press **Ctrl** and **a** at the same time, followed by **x**).



## 3. 启动：

###  1  Makefile里面出现的启动参数

见：Makefile 文件, 详情请参考 2.2节

```Makefile
QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```



启动参数的简单描述：

> 1. `-machine virt`: 指定使用 "virt" 机器类型，这是一种通用的虚拟机配置，适用于多种操作系统和应用程序。
> 2. `-bios none`: 禁用 BIOS，这意味着模拟器将不会加载任何 BIOS。在此配置中，模拟器直接从内核启动。
> 3. `-kernel $K/kernel`: 指定内核文件的路径。
> 4. `-m 128M`: 分配给模拟器的内存大小为 128 MB。
> 5. `-smp $(CPUS)`: 指定模拟器中的处理器数量。
> 6. `-nographic`: 禁用图形化界面，以纯文本方式运行模拟器。
> 7. `-global virtio-mmio.force-legacy=false`: 设置 virtio-mmio 总线为非遗留模式。virtio-mmio 是一种虚拟设备总线，用于连接虚拟机和宿主机之间的块设备和其他设备。
> 8. `-drive file=fs.img,if=none,format=raw,id=x0`: 添加一个虚拟硬盘驱动器，`fs.img` 是硬盘镜像文件的路径，`if=none` 表示不将其连接到默认接口，`format=raw` 指定硬盘镜像的格式为原始二进制，`id=x0` 给驱动器分配一个唯一的标识符。
> 9. `-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0`: 添加一个 virtio 块设备，将前面创建的驱动器连接到 virtio-mmio 总线上。`drive=x0` 表示将驱动器 `x0` 连接到该设备上，`bus=virtio-mmio-bus.0` 指定设备连接到 virtio-mmio 总线的第一个设备槽上。

这些启动参数用于配置 QEMU 模拟器的运行环境，包括内核、内存、处理器、虚拟硬盘等方面。




### 2  Net lab中网络驱动启动参数：

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
   
      具体格式为 `hostfwd=udp::主机端口-虚拟机IP:虚拟机端口`。这里的 `虚拟机IP` 省略，默认转发到虚拟机的相应端口。
   
2. `-object filter-dump,id=net0,netdev=net0,file=packets.pcap`
   1. `-object filter-dump`: 配置一个数据包捕获过滤器对象。
   2. `id=net0`: 为该对象分配一个 ID。
   3. `netdev=net0`: 指定与 `-netdev` 中配置的网络设备关联。
   4. `file=packets.pcap`: 指定捕获的数据包输出到 `packets.pcap` 文件中。这个文件将包含通过 `net0` 网络设备的所有数据包。
   
3. `-device e1000,netdev=net0,bus=pcie.0`
   1. `-device e1000`: 启动一个虚拟网络设备，类型为 `e1000`（Intel的一个常见网络接口卡模型）。
   2. `netdev=net0`: 将该设备绑定到先前定义的 `net0` 网络设备。
   3. `bus=pcie.0`: 指定该网络设备连接到的总线类型和位置。这里是 `pcie.0`，表示连接到 PCI Express 总线。



## 4. gdb调试工具介绍：

`gdb-multiarch`是GNU Debugger (GDB) 的一种不同变体，

### 1.  gdb-multiarch (调试时主要使用)

`gdb-multiarch` 是一种支持多种体系结构的GDB版本。可以用单一的GDB实例来调试多种不同体系结构的程序。适合处理多种硬件平台上的软件。而且也有一定灵活性，支持用户在同一个调试器中切换不同的体系结构目标。



