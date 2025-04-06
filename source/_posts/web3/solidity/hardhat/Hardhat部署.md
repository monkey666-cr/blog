---
title: "Hardhat部署"
date: {{ date }}
categories:
    - Web3
    - Solidity
    - Hardhat
---

## 摘要

在学习了chainlink的Price Feed功能之后，一直是将合约部署在Sepolia测试网上进行测试，每次部署耗时，不方便，所以本篇记录一下如何使用Hardhat的ignition功能在local网络进行部署。参考[教学视频FundMe合约](https://github.com/smartcontractkit/full-blockchain-solidity-course-js#lesson-7-hardhat-fund-me)进行改造。

## 环境

- Solidity
- Hardhat
- Chainlink
- FundMe Contract

## 创建项目

```bash
# 创建项目
mkdir hardhat-fund-me-fcc
cd hardhat-fund-me
# 初始化项目
yarn init -y
# 安装依赖
yarn add --dev hardhat
# 初始化hardhat项目
yarn hardhat init
```

## 移植FundMe合约

导入失败问题，新版本的AggregatorV3Interface.sol文件路径发生了改变。`import "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
`

## Chainlink Price Feed Mock

```solidity
// 文件路径: contracts/tests/MockV3Aggregator.sol

// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "@chainlink/contracts/src/v0.8/tests/MockV3Aggregator.sol";
```

## 合约脚本

### 创建配置文件`helper-hardhat-config.js`

```js
const networkConfig = {
  31337: {
    name: "localhost",
  },
  // Price Feed Address, values can be obtained at https://docs.chain.link/data-feeds/price-feeds/addresses
  11155111: {
    name: "sepolia",
    ethUsdPriceFeed: "0x694AA1769357215DE4FAC081bf1f309aDC325306",
  },
};

const developmentChains = ["hardhat", "localhost"];

module.exports = {
  networkConfig,
  developmentChains,
};
```

### 创建Mock合约的部署脚本`./ignition/modules/MockV3Aggregator.js`

```js
const { network } = require("hardhat");
const { buildModule } = require("@nomicfoundation/hardhat-ignition/modules");
const { developmentChains } = require("../../helper.hardhat-config");

const DECIMALS = "8";
const INITIAL_ANSWER = "200000000000";

const deployV3Aggregator = buildModule("deployV3AgggregatorModule", (m) => {
  if (developmentChains.includes(network.name)) {
    const priceFeed = m.contract("MockV3Aggregator", [
      DECIMALS,
      INITIAL_ANSWER,
    ]);
    return { priceFeed };
  }
});

module.exports = { deployV3Aggregator };
```

### 创建FundMe合约的部署脚本`./ignition/modules/FundMe.js`

```js
const { network } = require("hardhat");
const { buildModule } = require("@nomicfoundation/hardhat-ignition/modules");
const {
  developmentChains,
  networkConfig,
} = require("../../helper.hardhat-config");
const { deployV3Aggregator } = require("./MockV3Aggregator");

module.exports = buildModule("FundMeModule", (m) => {
  let priceFeed;
  if (developmentChains.includes(network.name)) {
    priceFeed = m.useModule(deployV3Aggregator).priceFeed;
  } else {
    priceFeed = m.contractAt(
      "AggregatorV3Interface",
      networkConfig[network.config.chainId].ethUsdPriceFeed
    );
  }
  const fundMe = m.contract("FundMe", [priceFeed]);

  return { fundMe };
});
```

### `hardhat.config.js`

```js
require("@nomicfoundation/hardhat-toolbox");
require("@nomicfoundation/hardhat-verify");
require("dotenv").config();

const SEPOLIA_RPC_URL = process.env.SEPOLIA_RPC_URL;
const PRIVATE_KEY = process.env.PRIVATE_KEY;
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY;

const LOCAL_RPC_URL = process.env.LOCAL_RPC_URL;
const LOCAL_PRIVATE_KEY = process.env.LOCAL_PRIVATE_KEY;

const COINMARKETCAP_API_KEY = process.env.COINMARKETCAP_API_KEY;
console.log(COINMARKETCAP_API_KEY);

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.28",
  etherscan: {
    apiKey: ETHERSCAN_API_KEY,
  },
  networks: {
    local: {
      url: LOCAL_RPC_URL,
      accounts: [LOCAL_PRIVATE_KEY],
      chainId: 31337,
    },
    sepolia: {
      url: SEPOLIA_RPC_URL,
      accounts: [PRIVATE_KEY],
      chainId: 11155111,
    },
  },
  gasReporter: {
    enabled: true,
    currency: "USD",
    outputFile: "gas-report.txt",
    noColors: true,
    coinmarketcap: COINMARKETCAP_API_KEY,
    token: "ETH",
    offline: true,
    tokenPrice: 0.000540541,
    gasPrice: 413000000,
  },
};
```

## 部署合约

本地网络部署

```bash
yarn hardhat ignition deploy ./ignition/modules/FundMe.js
```

测试环境部署

```bash
yarn hardhat ignition deploy ./ignition/modules/FundMe.js --network sepolia
```

## 总结

实践了Hardhat的ignition功能，在本地网络Mock Chainlink Price Feed，在测试网络部署合约。