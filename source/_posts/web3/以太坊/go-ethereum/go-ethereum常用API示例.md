---
title: "go-ethereum常用API示例"
date: {{ date }}
categories:
    - Web3
    - 以太坊
    - go-ethereum
---

## 摘要

学习go-ethereum与区块链的交互

## 查询区块

如果未传`blockNumber`则返回最新的区块头

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	client, err := ethclient.Dial("https://eth-sepolia.g.alchemy.com/v2/<API key>")
	if err != nil {
		log.Fatal(err)
	}

	blockNumber := big.NewInt(5671744)
	header, err := client.HeaderByNumber(context.Background(), nil)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(header.Number.Uint64())
	fmt.Println(header.Time)
	fmt.Println(header.Difficulty.Uint64())
	fmt.Println(header.Hash().Hex())

	block, err := client.BlockByNumber(context.Background(), blockNumber)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(block.Number().Uint64())
	fmt.Println(block.Time())
	fmt.Println(block.Difficulty().Uint64())
	fmt.Println(block.Hash().Hex())
	fmt.Println(len(block.Transactions()))
	count, err := client.TransactionCount(context.Background(), block.Hash())
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(count)
}
```

## 查询交易

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	client, err := ethclient.Dial("https://eth-sepolia.g.alchemy.com/v2/<API key>")
	if err != nil {
		log.Fatal(err)
	}

	chainID, err := client.ChainID(context.Background())
	if err != nil {
		log.Fatal(err)
	}
	blockNumber := big.NewInt(5671744)
	block, err := client.BlockByNumber(context.Background(), blockNumber)
	if err != nil {
		log.Fatal(err)
	}

	for _, tx := range block.Transactions() {
		fmt.Println(tx.Hash().Hex())
		fmt.Println(tx.Value().String())
		fmt.Println(tx.Gas())
		fmt.Println(tx.GasPrice().Uint64())
		fmt.Println(tx.Nonce())
		fmt.Println(tx.Data())
		fmt.Println(tx.To().Hex())

		if sender, err := types.Sender(types.NewEIP155Signer(chainID), tx); err == nil {
			fmt.Println(sender.Hex())
		} else {
			log.Fatal(err)
		}

		receipt, err := client.TransactionReceipt(context.Background(), tx.Hash())
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println(receipt.Status)
		fmt.Println(receipt.Logs)

		break
	}

	blockHash := common.HexToHash("0xae713dea1419ac72b928ebe6ba9915cd4fc1ef125a606f90f5e783c47cb1a4b5")
	count, err := client.TransactionCount(context.Background(), blockHash)
	if err != nil {
		log.Fatal(err)
	}
	for idx := uint(0); idx < count; idx++ {
		tx, err := client.TransactionInBlock(context.Background(), blockHash, idx)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println(tx.Hash().Hex())
		break
	}

	txHash := common.HexToHash("0x20294a03e8766e9aeab58327fc4112756017c6c28f6f99c7722f4a29075601c5")
	tx, isPending, err := client.TransactionByHash(context.Background(), txHash)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(isPending)
	fmt.Println(tx.Hash().Hex())
}
```

## 查询收据

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/rpc"
)

func main() {
	client, err := ethclient.Dial("https://eth-sepolia.g.alchemy.com/v2/<API key>")
	if err != nil {
		log.Fatal(err)
	}

	blockNumber := big.NewInt(5671744)
	blockHash := common.HexToHash("0xae713dea1419ac72b928ebe6ba9915cd4fc1ef125a606f90f5e783c47cb1a4b5")

	receiptByHash, err := client.BlockReceipts(context.Background(), rpc.BlockNumberOrHashWithHash(blockHash, false))
	if err != nil {
		log.Fatal(err)
	}

	receiptsByNum, err := client.BlockReceipts(context.Background(), rpc.BlockNumberOrHashWithNumber(rpc.BlockNumber(blockNumber.Int64())))
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(receiptByHash[0] == receiptsByNum[0])

	for _, receipt := range receiptByHash {
		fmt.Println(receipt.Status)
		fmt.Println(receipt.Logs)
		fmt.Println(receipt.TxHash.Hex())
		fmt.Println(receipt.TransactionIndex)
		fmt.Println(receipt.ContractAddress.Hex())
		break
	}

	txHash := common.HexToHash("0x20294a03e8766e9aeab58327fc4112756017c6c28f6f99c7722f4a29075601c5")
	receipt, err := client.TransactionReceipt(context.Background(), txHash)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(receipt.Status)
	fmt.Println(receipt.Logs)
	fmt.Println(receipt.TxHash.Hex())
	fmt.Println(receipt.TransactionIndex)
	fmt.Println(receipt.ContractAddress.Hex())
}
```

## 创建新钱包

```golang
package main

import (
	"crypto/ecdsa"
	"fmt"
	"log"

	"github.com/ethereum/go-ethereum/common/hexutil"
	"github.com/ethereum/go-ethereum/crypto"
	"golang.org/x/crypto/sha3"
)

func main() {
	privateKey, err := crypto.GenerateKey()
	if err != nil {
		log.Fatal(err)
	}

	privateKeyBytes := crypto.FromECDSA(privateKey)
	fmt.Println(hexutil.Encode(privateKeyBytes)[2:])
	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal(err)
	}

	publicKeyBytes := crypto.FromECDSAPub(publicKeyECDSA)
	fmt.Println("from pubkey:", hexutil.Encode(publicKeyBytes)[4:])
	address := crypto.PubkeyToAddress(*publicKeyECDSA).Hex()
	fmt.Println(address)
	hash := sha3.NewLegacyKeccak256()
	hash.Write(publicKeyBytes[1:])
	fmt.Println("full:", hexutil.Encode(hash.Sum(nil)[:]))
	fmt.Println(hexutil.Encode(hash.Sum(nil)[12:]))
}
```

## ETH转账

```golang
package main

import (
	"context"
	"crypto/ecdsa"
	"fmt"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	client, err := ethclient.Dial("https://eth-sepolia.g.alchemy.com/v2/<API key>")
	if err != nil {
		log.Fatal(err)
	}
	privateKey, err := crypto.HexToECDSA("")
	if err != nil {
		log.Fatal(err)
	}

	publicKey := privateKey.Public()
	publicKeyECDS, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("cannot assert type: pupblicKey is not of type *ecsa.PublicKey")
	}
	fromAddress := crypto.PubkeyToAddress(*publicKeyECDS)
	fmt.Println(fromAddress)
	nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(nonce)

	value := big.NewInt(1000000000000000000)
	gasLimit := uint64(21000)
	gasPrice, err := client.SuggestGasPrice(context.Background())
	if err != nil {
		log.Fatal(err)
	}

	toAddress := common.HexToAddress("0x220936506Deb81D789860013d3b2CD46f9bD7D15")
	var data []byte
	tx := types.NewTransaction(nonce, toAddress, value, gasLimit, gasPrice, data)

	chainID, err := client.NetworkID(context.Background())
	if err != nil {
		log.Fatal(err)
	}

	signedTx, err := types.SignTx(tx, types.NewEIP155Signer(chainID), privateKey)
	if err != nil {
		log.Fatal(err)
	}

	err = client.SendTransaction(context.Background(), signedTx)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("tx sent: %s", signedTx.Hash().Hex())
}
```

## 代币转账

```golang
package main

import (
	"context"
	"crypto/ecdsa"
	"fmt"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/common/hexutil"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
	"golang.org/x/crypto/sha3"
)

func main() {
	client, err := ethclient.Dial("https://eth-sepolia.g.alchemy.com/v2/<API key>")
	if err != nil {
		log.Fatal(err)
	}

	privateKey, err := crypto.HexToECDSA("")
	if err != nil {
		log.Fatal(err)
	}

	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("cannot assert type: publicKey is not of type *ecdsa.PublicKey")
	}

	fromAddress := crypto.PubkeyToAddress(*publicKeyECDSA)
	fmt.Println(fromAddress)

	nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
	if err != nil {
		log.Fatal(err)
	}

	value := big.NewInt(0)
	gasPrice, err := client.SuggestGasPrice(context.Background())
	if err != nil {
		log.Fatal(err)
	}

	toAddress := common.HexToAddress("0x220936506Deb81D789860013d3b2CD46f9bD7D15")
	tokenAddress := common.HexToAddress("0xAdE21088ff4EfC6e6756615865F75FDa34B773F7")

	transferFnSignature := []byte("transfer(address,uint256)")
	hash := sha3.NewLegacyKeccak256()
	hash.Write(transferFnSignature)
	methodId := hash.Sum(nil)[:4]
	fmt.Println(hexutil.Encode(methodId))
	paddedAddress := common.LeftPadBytes(toAddress.Bytes(), 32)
	fmt.Println(hexutil.Encode(paddedAddress))
	amount := new(big.Int)
	amount.SetString("1000000000000000000000", 10)
	paddedAmount := common.LeftPadBytes(amount.Bytes(), 32)
	fmt.Println(hexutil.Encode(paddedAmount))
	var data []byte
	data = append(data, methodId...)
	data = append(data, paddedAddress...)
	data = append(data, paddedAmount...)

	gasLimit, err := client.EstimateGas(context.Background(), ethereum.CallMsg{
		To:   &toAddress,
		Data: data,
	})
	if err != nil {
		log.Fatal(err)
	}

	gasLimit = gasLimit * 2
	fmt.Println(gasLimit)
	tx := types.NewTransaction(nonce, tokenAddress, value, gasLimit, gasPrice, data)

	chainID, err := client.NetworkID(context.Background())
	if err != nil {
		log.Fatal(err)
	}

	signedTx, err := types.SignTx(tx, types.NewEIP155Signer(chainID), privateKey)
	if err != nil {
		log.Fatal(err)
	}

	err = client.SendTransaction(context.Background(), signedTx)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("tx sent: %s", signedTx.Hash().Hex())
}
```

## 查询账户余额

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"math"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	client, err := ethclient.Dial("https://eth-sepolia.g.alchemy.com/v2/<API key>")
	if err != nil {
		log.Fatal(err)
	}

	account := common.HexToAddress("0x8df197edAf4C297e22B777fd2DDA04254B3C0a84")
	balance, err := client.BalanceAt(context.Background(), account, nil)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(balance)

	blockNumer := big.NewInt(5532993)
	balanceAt, err := client.BalanceAt(context.Background(), account, blockNumer)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(balanceAt)
	fbalance := new(big.Float)
	fbalance.SetString(balance.String())
	ethValue := new(big.Float).Quo(fbalance, big.NewFloat(math.Pow10(18)))
	fmt.Println(ethValue)
	pendingBalance, err := client.PendingBalanceAt(context.Background(), account)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(pendingBalance)
}
```

## 查询代币余额

```golang
package main
import (
        "fmt"
        "log"
        "math"
        "math/big"
        "github.com/ethereum/go-ethereum/accounts/abi/bind"
        "github.com/ethereum/go-ethereum/common"
        "github.com/ethereum/go-ethereum/ethclient"
        token "./contracts_erc20" // for demo
)
func main() {
        client, err := ethclient.Dial("https://cloudflare-eth.com")
        if err != nil {
                log.Fatal(err)
        }
        // Golem (GNT) Address
        tokenAddress := common.HexToAddress("0xfadea654ea83c00e5003d2ea15c59830b65471c0")
        instance, err := token.NewToken(tokenAddress, client)
        if err != nil {
                log.Fatal(err)
        }
        address := common.HexToAddress("0x25836239F7b632635F815689389C537133248edb")
        bal, err := instance.BalanceOf(&bind.CallOpts{}, address)
        if err != nil {
                log.Fatal(err)
        }
        name, err := instance.Name(&bind.CallOpts{})
        if err != nil {
                log.Fatal(err)
        }
        symbol, err := instance.Symbol(&bind.CallOpts{})
        if err != nil {
                log.Fatal(err)
        }
        decimals, err := instance.Decimals(&bind.CallOpts{})
        if err != nil {
                log.Fatal(err)
        }
        fmt.Printf("name: %s\n", name)         // "name: Golem Network"
        fmt.Printf("symbol: %s\n", symbol)     // "symbol: GNT"
        fmt.Printf("decimals: %v\n", decimals) // "decimals: 18"
        fmt.Printf("wei: %s\n", bal) // "wei: 74605500647408739782407023"
        fbal := new(big.Float)
        fbal.SetString(bal.String())
        value := new(big.Float).Quo(fbal, big.NewFloat(math.Pow10(int(decimals))))
        fmt.Printf("balance: %f", value) // "balance: 74605500.647409"
}
```

## 订阅区块

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/rpc"
)

func main() {
	// Sepolia 测试网的 WebSocket RPC 节点（可以使用 Infura 或 Alchemy）
	sepoliaWebSocketURL := "wss://eth-sepolia.g.alchemy.com/v2/<API key>" // 替换为你的 API Key

	// 创建 RPC 客户端
	client, err := rpc.Dial(sepoliaWebSocketURL)
	if err != nil {
		log.Fatalf("Failed to connect to Sepolia: %v", err)
	}
	defer client.Close()

	// 创建 ethclient
	ethClient := ethclient.NewClient(client)
	defer ethClient.Close()

	// 订阅新区块事件
	headers := make(chan *types.Header)
	sub, err := ethClient.SubscribeNewHead(context.Background(), headers)
	if err != nil {
		log.Fatalf("Failed to subscribe to new blocks: %v", err)
	}
	defer sub.Unsubscribe()

	fmt.Println("Listening for new blocks on Sepolia...")

	for {
		select {
		case err := <-sub.Err():
			log.Fatalf("Subscription error: %v", err)
		case header := <-headers:
			block, err := ethClient.BlockByHash(context.Background(), header.Hash())
			if err != nil {
				log.Printf("Failed to fetch block: %v", err)
				continue
			}
			fmt.Printf("New Block: #%d, Hash: %s, Transactions: %d\n",
				block.Number().Uint64(),
				block.Hash().Hex(),
				len(block.Transactions()),
			)
		}
	}
}
```

## 部署合约

## 加载合约

## 执行合约

## 合约事件