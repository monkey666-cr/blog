---
title: "linker.ld解析"
date: {{ date }}
categories:
    - 操作系统
    - 操作系统训练营
    - rCore-Tutorial
    - Level2
    - 番外篇
---

## 摘要

针对rCore-Tutorial-2025S的linker.ld文件进行解析。

## 代码

```asm
OUTPUT_ARCH(riscv)
ENTRY(_start)

BASE_ADDRESS = 0x0;

SECTIONS
{
    . = BASE_ADDRESS;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
    . = ALIGN(4K);
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    . = ALIGN(4K);
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    .bss : {
        start_bss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
        end_bss = .;
    }
    /DISCARD/ : {
        *(.eh_frame)
        *(.debug*)
    }
}
```

## 解析

这是一个用于RISC-V架构的链接脚本，它定义了程序的内存布局和各个段的组织方式。下面我将详细解释每个部分：

### 基本设置
```ld
OUTPUT_ARCH(riscv)
ENTRY(_start)
```
- `OUTPUT_ARCH(riscv)`：指定输出文件的目标架构为RISC-V
- `ENTRY(_start)`：指定程序的入口点为`_start`符号

### 内存布局
```ld
BASE_ADDRESS = 0x0;
```
- 定义程序的基地址为0x0，表示代码将从地址0开始加载

### 段(SECTION)定义
```ld
SECTIONS
{
    . = BASE_ADDRESS;
```
- 设置当前位置计数器为BASE_ADDRESS(0x0)

#### 文本段(.text)
```ld
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
```
- 首先包含所有`.text.entry`段（通常是入口代码）
- 然后包含所有其他`.text`和`.text.*`段（程序代码）
- 这个段从基地址0x0开始

#### 只读数据段(.rodata)
```ld
    . = ALIGN(4K);
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
```
- `ALIGN(4K)`：将当前位置对齐到4KB边界
- 包含所有只读数据段（`.rodata`, `.rodata.*`, `.srodata`, `.srodata.*`）

#### 数据段(.data)
```ld
    . = ALIGN(4K);
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
```
- 再次对齐到4KB边界
- 包含所有可读写数据段（`.data`, `.data.*`, `.sdata`, `.sdata.*`）

#### BSS段(.bss)
```ld
    .bss : {
        start_bss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
        end_bss = .;
    }
```
- 定义未初始化数据段
- `start_bss`和`end_bss`符号用于标记BSS段的开始和结束，通常在启动代码中用于清零BSS段
- 包含所有BSS段（`.bss`, `.bss.*`, `.sbss`, `.sbss.*`）

#### 丢弃的段
```ld
    /DISCARD/ : {
        *(.eh_frame)
        *(.debug*)
    }
}
```
- 丢弃不需要的段：
  - `.eh_frame`：异常处理框架信息（通常不需要在裸机程序中）
  - `.debug*`：所有调试信息（减少最终二进制文件大小）

## 特点总结
1. 简单清晰的布局：代码→只读数据→可读写数据→未初始化数据
2. 4KB对齐：关键段之间使用4KB对齐，适合分页内存管理
3. 明确的入口点：`_start`作为程序入口
4. BSS段标记：提供了清零BSS段的便利
5. 调试信息丢弃：减小最终二进制体积

这个链接脚本适用于裸机(bare-metal)或操作系统内核的RISC-V程序，它假设程序将从地址0开始运行，并且没有考虑动态链接等复杂情况。