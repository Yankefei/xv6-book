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

Makefile 是用于指定项目中文件之间依赖关系和如何编译这些文件的一种文件。下面是 Makefile 文件的基本语法：

**规则（Rules）**：Makefile 中最基本的元素是规则，它告诉 make 工具如何生成一个或多个目标文件。语法如下：

```Plain
target: dependencies
    command
```



## 3. 细分功能:

### 1. Tags 功能

```Makefile
tags: $(OBJS) _init
  etags *.S *.c
```

在 Makefile 中，这句语句的作用是生成代码浏览的标签文件。让我们逐步解释：

- `tags: $(OBJS) _init`: 这是一个目标规则，其中`tags`是目标名称，`(OBJS) _init`是依赖项列表。这意味着要生成名为`tags`的目标文件，其依赖于`$(OBJS)`和`_init`这些文件或目标。
- `etags *.S *.c`: 这是生成标签文件的命令。`etags`是一个工具程序，用于创建代码浏览器所需的标签文件。通常，`*.S`和`*.c`通配符表示所有的汇编语言文件和C语言文件。

因此，当运行`make tags`时，Makefile将检查`$(OBJS)`和`_init`是否需要更新，如果有任何依赖项需要更新，则执行`etags *.S *.c`命令来生成标签文件`tags`。**生成的标签文件**可以用于代码导航和快速定位特定函数或变量的定义和引用。

### 2. 模式规则

```Makefile
ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o

_%: %.o $(ULIB)
  $(LD) $(LDFLAGS) -T $U/user.ld -o $@ $^
  $(OBJDUMP) -S $@ > $*.asm
  $(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym
```

上面语句的解析：

> 这段 Makefile 包含了一个模式规则，用于生成用户程序的可执行文件、汇编代码和符号表。
>
> - `ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o`: 定义了一个变量 `ULIB`，包含了多个目标文件，这些目标文件是用户库的一部分。
> - `_%: %.o $(ULIB)`: 这是一个模式规则，指定了如何生成名为`_XXX`的可执行文件，其中`XXX`是对应的`.o`文件的名称。依赖项包括当前目录下的`.o`文件以及定义的`ULIB`中的目标文件。
> - `$(LD) $(LDFLAGS) -T $U/user.ld -o $@ $^`: 使用链接器将目标文件和用户库链接在一起，生成可执行文件。
> - `$(OBJDUMP) -S $@ > $*.asm`: 使用`objdump`工具生成可执行文件的反汇编代码，并将结果输出到以当前文件名为基础的`.asm`文件中。
> - `$(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym`: 使用`objdump`工具提取可执行文件的符号表信息，并通过`sed`命令对其进行处理，然后将处理后的结果输出到以当前文件名为基础的`.sym`文件中。
>
> 因此，当执行类似`make XXX`的命令时，Makefile会根据对应的`.o`文件和用户库文件生成可执行文件、汇编代码文件和符号表文件。
>
> 注意： _%: %.o $(ULIB)  这样写，模式规则匹配时：生成的 xx 就不会包含  ULIB里面的文件了

`**.asm 文件非常重要，后面需要用它在lab debug中追踪当前代码执行的位置**`



### 3. 保留过程的生成物

```Makefile
.PRECIOUS: %.o
```

`.PRECIOUS: %.o` 这句 Makefile 表示将 `.o` 文件标记为“宝贵”的，即使在 Make 过程中被认为是临时文件并被删除的情况下，这些 `.o` 文件也会被保留下来。通常情况下，Make 工具会在生成目标文件后删除中间文件（比如 `.o` 文件），以保持工作目录的清洁。

但是，通过在 Makefile 中使用 `.PRECIOUS: %.o`，你告诉 Make 工具不要删除任何匹配模式`%.o`的中间文件。这可以确保在 Make 过程中即使出现意外终止或其他问题，中间文件也会被保留下来，以便进一步调试或分析问题。这在需要手动查看或检查中间文件时非常有用。



### 4. 按规则生成文件的方式

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

> 这段 Makefile 片段定义了一个变量 `UPROGS`，其中包含了一系列用户程序的路径。然后定义了一个规则，用于生成文件系统镜像 `fs.img`。
>
> - `UPROGS`: 这个变量包含了一系列用户程序的路径，每个路径代表一个用户程序。
> - `fs.img: mkfs/mkfs README $(UPROGS)`: 这是一个规则，指定了生成 `fs.img` 文件的依赖关系。即在生成 `fs.img` 文件之前，需要确保 `mkfs/mkfs` 可执行文件、`README` 文件以及 `UPROGS` 中定义的所有用户程序都是最新的。
> - `mkfs/mkfs fs.img README $(UPROGS)`: 这是规则的命令部分，指定了如何生成 `fs.img` 文件。它调用了 `mkfs/mkfs` 可执行文件，传递了 `fs.img` 和 `README` 作为参数，以及 `UPROGS` 中定义的所有用户程序。这个命令可能是在构建一个文件系统镜像的过程中使用用户程序。
>
> 因此，当执行 `make fs.img` 命令时，Make 工具会检查 `mkfs/mkfs` 可执行文件、`README` 文件和所有定义在 `UPROGS` 中的用户程序是否需要更新，如果有任何更改，则会调用相应的命令来重新生成 `fs.img` 文件。



### 5. 包含需要过程文件

```Makefile
-include kernel/*.d user/*.d 
```

通过 `-include` 命令，Make 工具会尝试包含指定的文件，如果文件存在则会被包含进来，如果文件不存在或无法读取，则会忽略错误继续执行。这对于自动生成的依赖关系文件特别有用，因为这些文件可能在一开始并不存在，但随着代码的编译过程会逐步生成。

因此，`-include kernel/*.d user/*.d` 会在 Make 执行时将 `kernel` 目录和 `user` 目录下的所有 `.d` 文件包含进来，以确保 Make 工具能够根据正确的依赖关系来决定何时需要重新编译源文件。