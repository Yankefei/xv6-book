# 2.1 xv6-编译-链接-脚本分析

## 1. 编译：

### 1.1 MakeFile 部分

参考：（todo）



### 1.2 链接脚本：

#### kernel.ld 文件分析

```C
OUTPUT_ARCH( "riscv" )

// 重要：ENTRY(_entry) 这句代码表示了内核的入口地址。
// 这个指令告诉链接器（ld）将 _entry 标签所代表的地址作为内核代码的入口点。
ENTRY( _entry )

SECTIONS
{
  /*
   * ensure that entry.S / _entry is at 0x80000000,
   * where qemu's -kernel jumps.
   */
  . = 0x80000000;

  // 这段是对 kernel里面的代码段进行处理
  .text : {
    // 将所有输入文件中出现的 .text 和 .text.* 节合并到输出文件的 .text 节中
    *(.text .text.*)
    // 在这里以0x1000 进行一次对齐
    . = ALIGN(0x1000);
    // 将当前位置（.）的值赋给了 _trampoline
    _trampoline = .;
    // 将 trampsec 节的内容追加到输出文件的当前位置
    // 注意，这里会将这个代码端重新追加到这里，
    //  .section trampsec 这个trampsec 定义在 一个汇编文件 trampoline.S中
    // 这样做可以保证后面 trampsec 的这个符号地址是对齐0x1000的
    // 实际打印：0x0000000080007000，确实是对齐的地址
    *(trampsec)
    // 重新对齐0x1000
    . = ALIGN(0x1000);
    // 对 _trampoline符号进行校验
    ASSERT(. - _trampoline == 0x1000, "error: trampoline larger than one page");
    // 在链接脚本（ld链接器的控制脚本）中，PROVIDE关键字用于定义一个符号，但仅当该符号尚未定义时。
    // 这意味着如果程序中没有其他地方定义了这个符号，链接器会根据PROVIDE指令定义它，
    // 如果程序中已经定义了这个符号，则PROVIDE指令不会覆盖已有的定义。
    PROVIDE(etext = .);  // 0x0000000080008000,
  }

  .rodata : {
    . = ALIGN(16);
    *(.srodata .srodata.*) /* do not need to distinguish this from .rodata */
    . = ALIGN(16);
    *(.rodata .rodata.*)
  }

  .data : {
    . = ALIGN(16);
    *(.sdata .sdata.*) /* do not need to distinguish this from .data */
    . = ALIGN(16);
    *(.data .data.*)
  }

  .bss : {
    . = ALIGN(16);
    *(.sbss .sbss.*) /* do not need to distinguish this from .bss */
    . = ALIGN(16);
    *(.bss .bss.*)
  }

  PROVIDE(end = .);  // 0x0000000080022270
}
```

#### 引用资料：

> ld 后缀的文件通常是链接器脚本文件，用于指导链接器如何将目标文件链接成最终的可执行文件或共享库文件。 在 ld 后缀的文件中，你可以定义各种链接器需要的规则和指令，例如定义程序入口点、指定内存布局、定义符号等。 ld 文件通常使用类似于以下的语法：

```Python
OUTPUT_FORMAT(格式)  # 指定输出文件的格式
OUTPUT_ARCH(架构)  # 指定输出文件的架构
ENTRY(入口点)  # 指定程序入口地址
SEARCH_DIR(搜索路径)  # 指定库文件的搜索路径
SECTIONS {  # 定义各个段的属性和内容
    .text : { *(.text) }  # 将所有.text节合并到输出文件的.text段
    .data : { *(.data) }  # 将所有.data节合并到输出文件的.data段
    ...
}
```



##### A. ENTRY(_entry) 语句

`ENTRY(_entry)` 指定了链接器在链接过程中将 `_entry` 标签所代表的地址设为内核代码的入口地址，作为内核代码的入口点, 也就是操作系统内核在启动时将会从这个地址开始执行。

##### B. SECTIONS 段

在内核的链接脚本文件（ld 文件）中，`SECTIONS` 段用于定义如何将输入文件（如目标文件、库文件等）的各个部分（sections）映射到输出文件（可执行文件）的各个段（segments）中，用来控制内存布局和段的分配。
