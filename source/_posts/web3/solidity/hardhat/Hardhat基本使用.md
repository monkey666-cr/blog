---
title: "Hardhat基本使用"
date: {{ date }}
categories:
    - Web3
    - Solidity
    - Hardhat
---

## 摘要

记录一下Hardhat框架的基本使用

## 环境

- Solidity
- Hardhat v2.22.19
- yarn v1.22.22

## 安装依赖

```bash
yarn add --dev hardhat
```

## 初始化项目


```bash
yarn hardhat init
```

```txt
$ /workspace/web3/hardhat-demo/node_modules/.bin/hardhat init
888    888                      888 888               888
888    888                      888 888               888
888    888                      888 888               888
8888888888  8888b.  888d888 .d88888 88888b.   8888b.  888888
888    888     "88b 888P"  d88" 888 888 "88b     "88b 888
888    888 .d888888 888    888  888 888  888 .d888888 888
888    888 888  888 888    Y88b 888 888  888 888  888 Y88b.
888    888 "Y888888 888     "Y88888 888  888 "Y888888  "Y888

Welcome to Hardhat v2.22.19

? What do you want to do? … 
▸ Create a JavaScript project
  Create a TypeScript project
  Create a TypeScript project (with Viem)
  Create an empty hardhat.config.js
  Quit
```

选择JavaScript project, 全部按照默认推荐的配置确认，完成项目的初始化

## 编译代码

```bash
yarn hardhat compile
```

## 执行单元测试

```bash
yarn hardhat test
```

## 启动测试节点

```bash
yarn hardhat node
```

## 部署合约

```bash
yarn hardhat ignition deploy ./ignition/modules/Lock.js --network localhost
```

