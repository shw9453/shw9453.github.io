---
title: "Balsn CTF 2020 - IdleGame"
categories:
  - blog
tags:
  - ctf
  - smart contract
---

For Balsn CTF 2020 online, I also created two smart-contract challenges, `Election` and `IdleGame`. The source files of them are available on [GitHub - shw9453/my-ctf-challenges](https://github.com/shw9453/my-ctf-challenges). Here is a walkthrough of the challenge `IdleGame`.

# Challenge

> Thinking about making money by playing games? Try the first idle game on the blockchain!

- Type: Smart contract
- Solves: 4/490
- Keywords: Continuous token, Flash-mintable token, Arbitrage, DeFi attack

# Solution

## Overview of the Game Contract

The `IdleGame` contract is designed as a real idle game, where players wait a specific time to earn `IDL` tokens and spend them on leveling up. Moreover, players can also buy and sell `IDL` tokens with `BSN` tokens. To reach our goal of earning a large number of `IDL` tokens, playing the game is not enough. We have to exploit the token contract's two features: flash-mintable and continuous supply.

## Flash-Mintable Tokens

[Flash-mintable](https://github.com/Austin-Williams/flash-mintable-tokens) tokens are ERC20-compliant tokens that allow flash-minting. Anyone can mint an arbitrary number of tokens as long as he can burn the same number of tokens at the end of the transaction, or the transaction reverts otherwise. Keep in mind that, with the flash-mintable capability, one can manipulate the total supply of a token to an arbitrarily large value as he desires.

## Continuous Tokens

If you are not familiar with continuous tokens, bounding curves or the Bancor formula, [here](https://yos.io/2018/11/10/bonding-curves/) is an excellent article to start. Recall that the Bancor formula determines the price of a continuous token as follows:

```
ContinuousTokenPrice = ReserveBalance / (ContinuousTokenSupply * ReserveRatio)
```

In this challenge, `IDL` is a continuous token with `BSN` as its reserve token, and the reserve ratio is 99.9% ([line 86](https://github.com/shw9453/my-ctf-challenges/blob/master/balsn-ctf-2020/IdleGame/public/IdleGame.sol#L86)). When buying tokens, both `ReserveBalance` and `ContinuousTokenSupply` increase, and so does the ContinuousTokenPrice. Selling tokens, on the other hand, reduces the price.

## Arbitraging by Manipulating the Token Price

Now comes the tricky part. How would the price of `IDL` change if we can directly control its total supply without minting or burning? According to the formula, the token price decreases as the total supply increases and vice versa. By controlling the total supply, we can perform an arbitrage attack.

First, flash-mint a large number of `IDL` tokens to raise the total supply and buy them with `BSN` tokens before the flash-minted tokens are burned. Second, sell all `IDL` tokens for `BSN` tokens. Since we buy the `IDL` tokens at a low price and sell them at a higher price, we get more `BSN` tokens than before. Repeat the attack multiple times to earn more `BSN` tokens until we have enough to exchange them to `IDL` and buy a flag.

# Misc

This challenge is inspired by the [Eminence attack](https://sampriestley.com/defi-arbs-explained-15m-eminence-attack/), where the attacker earned 15M USD (but also returned 8M). In this real-world attack, the total supply of the continuous token, `EMN`, was manipulated by buying and selling `eAAVE` tokens instead of using the flash-minting feature.

Thanks to [@aaditya_purani](https://github.com/perfectblue/ctf-writeups/tree/master/2020/BalsnCTF/IdleGame) and [@whysw_p](https://hackmd.io/@whysw/HJcnpCVqv) for their fantastic write-ups for this challenge. Hope you all enjoyed it!

Flag: `BALSN{Arb1tr4ge_wi7h_Fl45hMin7}`
