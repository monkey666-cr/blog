---
title: "rCore实验0: 实验环境配置"
date: {{ date }}
categories:
    - 操作系统
    - 操作系统训练营
    - rCore-Tutorial
    - Level2
---

## 摘要：

开始进行操作系统训练营Level2的实验，以下记录环境搭建过程。

## 实验环境

- Ubuntu 24.04.2 LTS
- Rust   1.85.1
- Cargo  1.85.1
- Qemu   7.0.0

## 安装Qemu

```bash
# 安装依赖
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev git tmux python3

# 下载qemu源码
wget https://download.qemu.org/qemu-7.0.0.tar.xz
# 解压
tar xvJf qemu-7.0.0.tar.xz
# 编译安装并配置 RISC-V 支持
cd qemu-7.0.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc) 
```

编辑`~/.bashrc`文件, 将编译好的二进制文件加入到PATH中, 并且执行`source ~/.bashrc`

```bash
# 注意 $HOME 是 Linux 自动设置的表示你家目录的环境变量，你也可以根据实际位置灵活调整
# 替换os-env
export PATH="$HOME/os-env/qemu-7.0.0/build/:$PATH"
export PATH="$HOME/os-env/qemu-7.0.0/build/riscv64-softmmu:$PATH"
export PATH="$HOME/os-env/qemu-7.0.0/build/riscv64-linux-user:$PATH"
```

确认Qemu的版本：

```bash
qemu-system-riscv64 --version
# qemu-riscv64 version 7.0.0
# Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers

qemu-riscv64 --version
# qemu-riscv64 version 7.0.0
# Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
```

## 试运行rCore-Tutorial

```bash
git clone https://github.com/LearningOS/rCore-Tutorial-Code-2025S
cd rCore-Tutorial-Code-2025S
```

切换到ch1分支

```bash
git checkout ch1
cd os
LOG=DEBUG make run
```

## 配置正确，得到如下输出

```txt
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
[DEBUG] [kernel] .rodata [0x80202000, 0x80203000)
[ INFO] [kernel] .data [0x80203000, 0x80204000)
[ WARN] [kernel] boot_stack top=bottom=0x80214000, lower_bound=0x80204000
[ERROR] [kernel] .bss [0x80214000, 0x80215000)
```