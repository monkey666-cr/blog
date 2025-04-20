---
title: "Hardhat配置"
date: {{ date }}
categories:
    - Web3
    - 以太坊
    - Solidity
    - Hardhat
---

## 摘要

上一篇内容记录了Hardhat的基本使用，本篇记录一下Hardhat的配置。如何配置测试网。


## hardhat.config.js代码

```js
require("@nomicfoundation/hardhat-toolbox");
require("@nomicfoundation/hardhat-verify");
require("dotenv").config();
require("./tasks/hello");

const SEPOLIA_RPC_URL = process.env.SEPOLIA_RPC_URL;
const PRIVATE_KEY = process.env.PRIVATE_KEY;
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY;

const LOCAL_RPC_URL = process.env.LOCAL_RPC_URL;
const LOCAL_PRIVATE_KEY = process.env.LOCAL_PRIVATE_KEY;

const COINMARKETCAP_API_KEY = process.env.COINMARKETCAP_API_KEY;

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

## .env配置

```env
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/
PRIVATE_KEY=
ETHERSCAN_API_KEY=

LOCAL_RPC_URL=http://127.0.0.1:8545/
LOCAL_PRIVATE_KEY=

COINMARKETCAP_API_KEY=
```

## package.json


```json
{
  "devDependencies": {
    "@nomicfoundation/hardhat-chai-matchers": "^2.0.0",
    "@nomicfoundation/hardhat-ethers": "^3.0.0",
    "@nomicfoundation/hardhat-ignition": "^0.15.0",
    "@nomicfoundation/hardhat-ignition-ethers": "^0.15.0",
    "@nomicfoundation/hardhat-network-helpers": "^1.0.0",
    "@nomicfoundation/hardhat-toolbox": "^5.0.0",
    "@nomicfoundation/hardhat-verify": "^2.0.0",
    "@typechain/ethers-v6": "^0.5.0",
    "@typechain/hardhat": "^9.0.0",
    "chai": "^4.2.0",
    "dotenv": "^16.4.7",
    "ethers": "^6.4.0",
    "hardhat": "^2.22.19",
    "hardhat-gas-reporter": "^2.2.2",
    "solidity-coverage": "^0.8.0",
    "typechain": "^8.3.0"
  }
}
```