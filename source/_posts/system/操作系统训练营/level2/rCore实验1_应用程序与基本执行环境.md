---
title: "rCore实验1: 应用程序与基本执行环境"
date: {{ date }}
categories:
    - 操作系统
    - 操作系统训练营
    - rCore-Tutorial
    - Level2
---

## 摘要

这一篇文章记录了如何实现打印`Hello, world!`的OS。

---

## 引言

为了完成实验目的，需要让程序不依赖标准库，并且通过编译。现在用户态完成代码，最终将程序移植到内核态。

### 实践体验

- 获取代码

```bash
$ git clone https://github.com/LearningOS/rCore-Tutorial-Code-2025S
$ cd rCore-Tutorial-Code-2025S
$ git checkout ch1
```

- 运行代码

```bash
cd os
make run LOG=TRACE
```

- 预期输出

```text

Platform: qemu
   Compiling os v0.1.0 (/home/chenrun/workspace/rust/2025s-rcore-monkey666-cr/os)
    Finished `release` profile [optimized + debuginfo] target(s) in 0.14s
[rustsbi] RustSBI version 0.3.0-alpha.4, adapting to RISC-V SBI v1.0.0
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
[rustsbi] Platform Name      : riscv-virtio,qemu
[rustsbi] Platform SMP       : 1
[rustsbi] Platform Memory    : 0x80000000..0x88000000
[rustsbi] Boot HART          : 0
[rustsbi] Device Tree Region : 0x87000000..0x87000ef2
[rustsbi] Firmware Address   : 0x80000000
[rustsbi] Supervisor Address : 0x80200000
[rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
[rustsbi] pmp02: 0x80000000..0x80200000 (---)
[rustsbi] pmp03: 0x80200000..0x88000000 (xwr)
[rustsbi] pmp04: 0x88000000..0x00000000 (-wr)
[kernel] Hello, world!
[TRACE] [kernel] .text [0x80200000, 0x80202000)
[DEBUG] [kernel] .rodata [0x80202000, 0x80203000)
[ INFO] [kernel] .data [0x80203000, 0x80204000)
[ WARN] [kernel] boot_stack top=bottom=0x80214000, lower_bound=0x80204000
[ERROR] [kernel] .bss [0x80214000, 0x80215000)
```

## 应用程序执行环境与平台支持

### 初始化Rust项目

```bash
make rCore-Level2-1
cd rCore-Level2-1
cargo init
```

![图片](/assets/images/system/操作系统训练营/level2/rCore实验1_应用程序与基本执行环境/初始化项目.png)


### 理解应用程序执行环境

![图片](/assets/images/system/操作系统训练营/level2/rCore实验1_应用程序与基本执行环境/程序执行环境.png)

<center>应用程序执行环境：图中的白色块自上而下表示各级执行环境，黑色块表示相邻两层执行环境之间的接口。</center>

### 平台与目标三元组

编译器在编译、链接得到可执行文件需要知道，程序要在那个**平台（Platform）**上运行，**目标三元组（Target Triplet）**描述了目标平台的CPU指令集、操作系统和标准运行时库。

```text
(base) chenrun:~/workspace/rust/rCore-Level2-1$ rustc --version --verbose
rustc 1.85.1 (4eb161250 2025-03-15)
binary: rustc
commit-hash: 4eb161250e340c8f48f66e2b929ef4a5bed7c181
commit-date: 2025-03-15
host: x86_64-unknown-linux-gnu
release: 1.85.1
LLVM version: 19.1.7
```

- host: `x86_64-unknown-linux-gnu`, CPU架构时`x86_64`，CPU厂商是`unknown`，操作系统是`linux`，运行库是`gnu libc`

目标是将程序移植到`RICV`平台`riscv64gc-unknown-none-elf`上运行。

> 注解
> <small>`riscv64gc-unknown-none-elf`的 CPU 架构是 riscv64gc，厂商是 unknown，操作系统是 none， elf 表示没有标准的运行时库。没有任何系统调用的封装支持，但可以生成 ELF 格式的执行程序。 我们不选择有 linux-gnu 支持的 riscv64gc-unknown-linux-gnu，是因为我们的目标是开发操作系统内核，而非在 linux 系统上运行的应用程序。</small>

### 修改目标平台

```bash
cargo run --target riscv64gc-unknown-none-elf

   Compiling rCore-Level2-1 v0.1.0 (/home/chenrun/workspace/rust/rCore-Level2-1)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not support the standard library
  = note: `std` is required by `rCore_Level2_1` because it does not declare `#![no_std]`
```


## 移除标准库依赖

在项目根目录下创建`.cargo`目录，并在这个目录下创建config文件

```text
# os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"
```

### 移除println!宏

```rust
// main.rs
#![no_std]

fn main() {
}
```

```bash
cargo build

error: `#[panic_handler]` function required, but not found
```

### 提供语义项`panic_handler`

新建子模块`lang_items.rs`，标记为`#[panic_handler]`

```rust
// src/lang_items.rs

use core::panic::{self, PanicInfo};

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

### vscode环境下修复编译报错的提示

![图片](/assets/images/system/操作系统训练营/level2/rCore实验1_应用程序与基本执行环境/no_std编译报错.png)

创建`.vscode/settings.json`，增加如下配置， 并重启rust-analyzer.cargo

```json
{
    "rust-analyzer.cargo.target": "riscv64gc-unknown-none-elf",
    "rust-analyzer.checkOnSave.allTargets": false
}
```

### 分析被移除标准库的程序

```bash
# 查看文件格式
file target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1

target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, with debug_info, not stripped

# 文件头信息
rust-readobj -h target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1

File: target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1
Format: elf64-littleriscv
Arch: riscv64
AddressSize: 64bit
......
  Type: Executable (0x2)
  Machine: EM_RISCV (0xF3)
  Version: 1
  Entry: 0x0
  ......

# 反汇编导出汇编程序
rust-objdump -S target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1

target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1: file format elf64-littleriscv
```

通过 file 工具对二进制程序 os 的分析可以看到，它好像是一个合法的 RV64 执行程序， 但 rust-readobj 工具告诉我们它的入口地址 Entry 是 0。 再通过 rust-objdump 工具把它反汇编，没有生成任何汇编代码。 可见，这个二进制程序虽然合法，但它是一个空程序，原因是缺少了编译器规定的入口函数 _start 。


## 构件用户态执行环境

### 用户态最小化执行环境

执行环境初始化

首先给Rust编译器提供入口函数`_start`，并在`main.rs`中添加以下内容

```rust
// src/main.rs

#[unsafe(no_mangle)]
extern "C" fn _start() {
    loop {}
}
```

重新编译, 编译器会自动生成`_start`函数，并把`_start`函数作为程序的入口地址。

```bash
cargo build

rust-objdump -S target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1

target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1: file format elf64-littleriscv

Disassembly of section .text:

0000000000011158 <_start>:
warning: address range table at offset 0x0 has a premature terminator entry at offset 0x10
;     loop {}
   11158: a009          j       0x1115a <_start+0x2>
   1115a: a001          j       0x1115a <_start+0x2>
```

反汇编出的两条指令就是一个死循环， 这说明编译器生成的已经是一个合理的程序了。 用 qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1 命令可以执行这个程序。

### 程序正常退出

现在将`_start`函数中的`loop()`注释掉，重新编译分析

```bash
rust-objdump -S target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1

target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1: file format elf64-littleriscv

Disassembly of section .text:

0000000000011158 <_start>:
warning: address range table at offset 0x0 has a premature terminator entry at offset 0x10
; }
   11158: 8082          ret
```

qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1 命令可以执行这个程序。会引发问题：
段错误 (核心已转储)

> 注解
> <samll>QEMU有两种运行模式：
> 
> User mode 模式，即用户态模拟，如 qemu-riscv64 程序， 能够模拟不同处理器的用户态指令的执行，并可以直接解析ELF可执行文件， 加载运行那些为不同处理器编译的用户级Linux应用程序。
> 
> System mode 模式，即系统态模式，如 qemu-system-riscv64 程序， 能够模拟一个完整的基于不同CPU的硬件系统，包括处理器、内存及其他外部设备，支持运行完整的操作系统。</small>

一下内容对我来讲就是魔法了，不懂汇编，嘻嘻。目前就先理解到`_start`函数调用了`sys_exit`函数，向操作系统发出退出的系统调用。代码如下：

```rust
// src/main.rs
#![no_std]
#![no_main]

const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id,
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

#[unsafe(no_mangle)]
extern "C" fn _start() {
    sys_exit(9);
}

mod lang_items;
```

重新编译，运行

```bash
qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/rCore-Level2-1; echo $?
9
```

### 有显示支持的用户态执行环境

Rust 的 core 库内建了以一系列帮助实现显示字符的基本 Trait 和数据结构，函数等，我们可以对其中的关键部分进行扩展，就可以实现定制的 println! 功能。

完整代码如下：

```rust
#![no_std]
#![no_main]

use core::fmt::{self, Write};

const SYSCALL_EXIT: usize = 93;
const SYSCALL_WRITE: usize = 64;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id,
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> core::fmt::Result {
        sys_write(1, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    };
}

#[unsafe(no_mangle)]
extern "C" fn _start() {
    println!("Hello world!");
    sys_exit(9);
}

mod lang_items;
```


## 构件裸机执行环境

### 裸机启动过程

用 QEMU 软件 qemu-system-riscv64 来模拟 RISC-V 64 计算机。加载内核程序的命令如下：

```bash
qemu-system-riscv64 \
            -machine virt \
            -nographic \
            -bios $(BOOTLOADER) \
            -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
```

- `-bios $(BOOTLOADER)` 指定引导程序，即 bootloader 程序， 它负责加载内核程序，并跳转到内核程序的入口地址。即 RustSBI。
- `-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)` 指定内核程序， 它负责初始化硬件设备，并启动操作系统。表示硬件内存中的特定位置。放置了操作系统的二进制代码 $(KERNEL_BIN) 。 $(KERNEL_ENTRY_PA) 的值是 0x80200000 。
- `-machine virt` 指定模拟的计算机类型为 virt， 即 QEMU 模拟的 RISC-V 64 计算机。
- `-nographic` 指定不使用图形界面， 即不使用显示器。

当我们执行包含上述启动参数的 qemu-system-riscv64 软件，就意味给这台虚拟的 RISC-V64 计算机加电了。 此时，CPU 的其它通用寄存器清零，而 PC 会指向 0x1000 的位置，这里有固化在硬件中的一小段引导代码， 它会很快跳转到 0x80000000 的 RustSBI 处。 RustSBI完成硬件初始化后，会跳转到 $(KERNEL_BIN) 所在内存位置 0x80200000 处， 执行操作系统的第一条指令。

![图片](/assets/images/system/操作系统训练营/level2/rCore实验1_应用程序与基本执行环境/裸机启动过程.png)

> 注解
> <small>SBI 是 RISC-V 的一种底层规范，RustSBI 是它的一种实现。 操作系统内核与 RustSBI 的关系有点像应用与操作系统内核的关系，后者向前者提供一定的服务。只是SBI提供的服务很少， 比如关机，显示字符串等。</small>

### 实现关机功能

```rust
// src/main.rs

#![no_std]
#![no_main]

/// general sbi call
#[inline(always)]
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "li x16, 0",
            "ecall",
            inlateout("x10") arg0 => ret,
            in("x11") arg1,
            in("x12") arg2,
            in("x17") which,
        );
    }
    ret
}

const SBI_SHUTDOWN: usize = 8;

pub fn shutdown() -> ! {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0);
    panic!("It should shutdown!");
}

mod lang_items;

core::arch::global_asm!(include_str!("entry.asm"));

fn clear_bss() {
    unsafe extern "C" {
        fn sbss();
        fn ebss();
    }
    unsafe {
        (sbss as usize..ebss as usize).for_each(|a| (a as *mut u8).write_volatile(0));
    }
}

#[unsafe(no_mangle)]
pub fn rust_main() -> ! {
    clear_bss();
    shutdown();
}
```

### 设置正确的程序内容布局

```ld
OUTPUT_ARCH(riscv)
ENTRY(_start)
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }

    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

配置文件中增加linker.ld文件路径

```txt
// .cargo/config

[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]
```


### 正确布置栈空间

增加entry.asm代码

```asm
    .section .text.entry
    .globl _start
_start:
    la sp, boot_stack_top
    call rust_main

    .section .bss.stack
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:
```

重新编译，运行

```bash
cargo build --release

rust-objcopy \
    --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/release/rCore-Level2-1 \
    --strip-all -O binary target/riscv64gc-unknown-none-elf/release/os.bin

qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios ./bootloader/rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
```

## 总结

至此完成了用户态应用程序的编写，并构建了用户态执行环境。完成裸机执行环境，并实现了关机功能。