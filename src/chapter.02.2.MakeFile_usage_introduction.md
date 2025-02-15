# 2.2 Makefile文件的分析和语法介绍


## 1. xv6 Makefile 文件解析：

```Makefile
K=kernel
U=user

OBJS = \
  $K/entry.o \
  $K/start.o \
  $K/console.o \
  $K/printf.o \
  $K/uart.o \
  $K/kalloc.o \
  $K/spinlock.o \
  $K/string.o \
  $K/main.o \
  $K/vm.o \
  $K/proc.o \
  $K/swtch.o \
  $K/trampoline.o \
  $K/trap.o \
  $K/syscall.o \
  $K/sysproc.o \
  $K/bio.o \
  $K/fs.o \
  $K/log.o \
  $K/sleeplock.o \
  $K/file.o \
  $K/pipe.o \
  $K/exec.o \
  $K/sysfile.o \
  $K/kernelvec.o \
  $K/plic.o \
  $K/virtio_disk.o

# riscv64-unknown-elf- or riscv64-linux-gnu-
# perhaps in /opt/riscv/bin
#TOOLPREFIX = 

# Try to infer the correct TOOLPREFIX if not set
ifndef TOOLPREFIX
TOOLPREFIX := $(shell if riscv64-unknown-elf-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
  then echo 'riscv64-unknown-elf-'; \
  elif riscv64-linux-gnu-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
  then echo 'riscv64-linux-gnu-'; \
  elif riscv64-unknown-linux-gnu-objdump -i 2>&1 | grep 'elf64-big' >/dev/null 2>&1; \
  then echo 'riscv64-unknown-linux-gnu-'; \
  else echo "***" 1>&2; \
  echo "*** Error: Couldn't find a riscv64 version of GCC/binutils." 1>&2; \
  echo "*** To turn off this error, run 'gmake TOOLPREFIX= ...'." 1>&2; \
  echo "***" 1>&2; exit 1; fi)
endif

QEMU = qemu-system-riscv64   # 

CC = $(TOOLPREFIX)gcc
AS = $(TOOLPREFIX)gas
LD = $(TOOLPREFIX)ld
OBJCOPY = $(TOOLPREFIX)objcopy   # 目标文件复制工具
OBJDUMP = $(TOOLPREFIX)objdump   # 目标文件反汇编工具

# 设置编译选项
CFLAGS = -Wall -Werror -O -fno-omit-frame-pointer -ggdb -gdwarf-2
CFLAGS += -MD
CFLAGS += -mcmodel=medany
CFLAGS += -ffreestanding -fno-common -nostdlib -mno-relax
CFLAGS += -I.
CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)

# Disable PIE when possible (for Ubuntu 16.10 toolchain)
ifneq ($(shell $(CC) -dumpspecs 2>/dev/null | grep -e '[^f]no-pie'),)
CFLAGS += -fno-pie -no-pie
endif
ifneq ($(shell $(CC) -dumpspecs 2>/dev/null | grep -e '[^f]nopie'),)
CFLAGS += -fno-pie -nopie
endif

LDFLAGS = -z max-page-size=4096

# 编译 kerne的时候，除了OBJS，还需要 kernel.ld 以及 initcode 
# 而initcode 则是由下面的 过程编译生成的，不含符号表，而且是二进制的形式
$K/kernel: $(OBJS) $K/kernel.ld $U/initcode
  $(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS) 
  $(OBJDUMP) -S $K/kernel > $K/kernel.asm
  $(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym

$U/initcode: $U/initcode.S
  $(CC) $(CFLAGS) -march=rv64g -nostdinc -I. -Ikernel -c $U/initcode.S -o $U/initcode.o
  $(LD) $(LDFLAGS) -N -e start -Ttext 0 -o $U/initcode.out $U/initcode.o
  $(OBJCOPY) -S -O binary $U/initcode.out $U/initcode
  $(OBJDUMP) -S $U/initcode.o > $U/initcode.asm

# 用于指导生成tag
tags: $(OBJS) _init
  etags *.S *.c

# usys.o 由下面的语句生成
ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o

# 这样写，生成的 xx 就不会包含 ULIB里面的文件了
_%: %.o $(ULIB)
  $(LD) $(LDFLAGS) -T $U/user.ld -o $@ $^
  $(OBJDUMP) -S $@ > $*.asm
  $(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym

# 通过脚本生成  usys.S
$U/usys.S : $U/usys.pl
  perl $U/usys.pl > $U/usys.S

# 再由 usys.S 生成 usys.o
$U/usys.o : $U/usys.S
  $(CC) $(CFLAGS) -c -o $U/usys.o $U/usys.S

$U/_forktest: $U/forktest.o $(ULIB)
  # forktest has less library code linked in - needs to be small
  # in order to be able to max out the proc table.
  $(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $U/_forktest $U/forktest.o $U/ulib.o $U/usys.o
  $(OBJDUMP) -S $U/_forktest > $U/forktest.asm

# mkfs 需要直接用gcc来编译，因为它需要在本地环境初始化一个镜像文件
mkfs/mkfs: mkfs/mkfs.c $K/fs.h $K/param.h
  gcc -Werror -Wall -I. -o mkfs/mkfs mkfs/mkfs.c

# Prevent deletion of intermediate files, e.g. cat.o, after first build, so
# that disk image (changes after first build) are persistent until clean.  More
# details:
# http://www.gnu.org/software/make/manual/html_node/Chained-Rules.html
.PRECIOUS: %.o

UPROGS=\
  $U/_cat\
  $U/_echo\
  $U/_forktest\
  $U/_grep\
  $U/_init\
  $U/_kill\
  $U/_ln\
  $U/_ls\
  $U/_mkdir\
  $U/_rm\
  $U/_sh\
  $U/_stressfs\
  $U/_usertests\
  $U/_grind\
  $U/_wc\
  $U/_zombie\

fs.img: mkfs/mkfs README $(UPROGS)
  mkfs/mkfs fs.img README $(UPROGS)

-include kernel/*.d user/*.d

clean: 
  rm -f *.tex *.dvi *.idx *.aux *.log *.ind *.ilg \
  */*.o */*.d */*.asm */*.sym \
  $U/initcode $U/initcode.out $K/kernel fs.img \
  mkfs/mkfs .gdbinit \
        $U/usys.S \
  $(UPROGS)

# try to generate a unique GDB port
GDBPORT = $(shell expr `id -u` % 5000 + 25000)
# QEMU's gdb stub command line changed in 0.11
QEMUGDB = $(shell if $(QEMU) -help | grep -q '^-gdb'; \
  then echo "-gdb tcp::$(GDBPORT)"; \
  else echo "-s -p $(GDBPORT)"; fi)
ifndef CPUS
CPUS := 3
endif

QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

qemu: $K/kernel fs.img
  $(QEMU) $(QEMUOPTS)

.gdbinit: .gdbinit.tmpl-riscv
  sed "s/:1234/:$(GDBPORT)/" < $^ > $@

qemu-gdb: $K/kernel .gdbinit fs.img
  @echo "*** Now run 'gdb' in another window." 1>&2
  $(QEMU) $(QEMUOPTS) -S $(QEMUGDB)
```



## 2. 基本规则

Makefile 是用于指定项目中文件之间依赖关系和如何编译这些文件的一种文件。

下面是 Makefile 文件的基本语法：它告诉 make 工具如何生成一个或多个目标文件。语法如下：

```Plain
target: dependencies
    command
```



## 3. 需要留意的细分功能:

### 1. 生成 asm文件和 sym文件

```Makefile
ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o

_%: %.o $(ULIB)
  $(LD) $(LDFLAGS) -T $U/user.ld -o $@ $^
  $(OBJDUMP) -S $@ > $*.asm
  $(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym
```

简单说明：

1. `ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o`: 定义了一个变量 `ULIB`，包含了多个目标文件，这些目标文件是用户库的一部分。

2. `_%: %.o $(ULIB)`: 这是一个模式规则，指定了如何生成名为`_XXX`的可执行文件，其中`XXX`是对应的`.o`文件的名称。依赖项包括当前目录下的`.o`文件以及定义的`ULIB`中的目标文件。

3. `$(LD) $(LDFLAGS) -T $U/user.ld -o $@ $^`: 使用链接器将目标文件和用户库链接在一起，生成可执行文件。

4. `$(OBJDUMP) -S $@ > $*.asm`: 使用`objdump`工具生成可执行文件的反汇编代码，并将结果输出到以当前文件名为基础的`.asm`文件中。

5. `$(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym`: 使用`objdump`工具提取可执行文件的符号表信息，并通过`sed`命令对其进行处理，然后将处理后的结果输出到以当前文件名为基础的`.sym`文件中。

因此，当执行类似`make XXX`的命令时，Makefile会根据对应的`.o`文件和用户库文件生成可执行文件、汇编代码文件和符号表文件。

**.asm 文件非常重要，后面需要用它在lab debug中追踪当前代码执行的汇编地址**



### 2.  fs.img文件生成

```Makefile
UPROGS=\
  $U/_cat\
  $U/_echo\
  $U/_forktest\
  $U/_grep\
  $U/_init\
  $U/_kill\
  $U/_ln\
  $U/_ls\
  $U/_mkdir\
  $U/_rm\
  $U/_sh\
  $U/_stressfs\
  $U/_usertests\
  $U/_grind\
  $U/_wc\
  $U/_zombie\

# 先指定文件的依赖项，然后再指定如何生成它
fs.img: mkfs/mkfs README $(UPROGS)
  mkfs/mkfs fs.img README $(UPROGS)
```

这段 Makefile 片段定义了一个变量 `UPROGS`，其中包含了一系列用户程序的路径，以及最终生成文件系统镜像 `fs.img`。

1. `fs.img: mkfs/mkfs README $(UPROGS)`: 这是一个规则，指定了生成 `fs.img` 文件的依赖关系。即在生成 `fs.img` 文件之前，需要确保 `mkfs/mkfs` 可执行文件、`README` 文件以及 `UPROGS` 中定义的所有用户程序都是最新的。

2. `mkfs/mkfs fs.img README $(UPROGS)`: 这是规则的命令部分，指定了如何生成 `fs.img` 文件：

   它调用了 `mkfs/mkfs` 可执行文件，传递了 `fs.img` 和 `README` 作为参数，以及 `UPROGS` 中定义的所有用户程序，后面mkfs 程序详解中，可以看到如何加载这些参数文件的，位于：8.9 节

