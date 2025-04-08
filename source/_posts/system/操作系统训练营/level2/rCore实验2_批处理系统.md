---
title: "rCore实验2: 批处理系统"
date: {{ date }}
categories:
    - 操作系统
    - 操作系统训练营
    - rCore-Tutorial
    - Level2
---

## 摘要

通过本次实验，了解了操作系统是如何执行任务的。如何在多任务之间进行切换。本次实验即将完成OS内核以批处理的形式运行多个应用程序，同时利用特权级机制，令OS不会因为出错的用户态程序而崩溃。

## 实现应用程序

记录一些知识点吧，加强一下印象。

### 应用程序设计

```rust
#[macro_use]
extern crate user_lib;
```

`user`目录下的代码是用户态程序，目录下的`lib.rs`以及引用的其他子模块，相当于编程语言提供的标准库。在`user/Cargo.toml`中对库的名字进行了设置：`name = "user_lib"`。为`bin`目录下的源程序提供依赖。

```rust
#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start() -> ! {
    clear_bss();
    exit(main());
}
```

使用`link_section`宏将`_start`函数编译后的汇编代码放在名为`.text.entry`的段中。方便用户库链接脚本将他作为用户程序入口。


```rust
#![feature(linkage)]

#[linkage = "weak"]
#[no_mangle]
fn main() -> i32 {
    panic!("main() should not be called.");
}
```

使用Rust宏将其标志为弱连接，这样最后连接的时候，虽然`lib.rs`和`bin`目录下的某个应用程序中都有`main`符号，但由于`lib.rs`中的`main`符号是弱连接，链接器会使用`bin`目录下的`main`符号。如果`bin`目录下找不到任何`main`，也能编译通过，但会在运行时报错。

### 系统调用

```rust
fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    usafe {
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
```

于是`sys_write`和`sys_exit`函数就变成了

```rust
const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}
```

将上述代码在`user_lib`中进一步封装。在`console`模块中，借助以下函数实现了`println!`宏。

```rust
use syscall::*;

pub fn write(fd: usize, buf: &[u8]) -> isize { sys_write(fd, buf) }
pub fn exit(exit_code: i32) -> isize { sys_exit(exit_code) }
```

## 实现批处理操作系统

## 实现特权级的切换

## 总结