---
title: "Balsn CTF 2020 - Election"
categories:
  - blog
tags:
  - ctf
  - smart contract
---

For Balsn CTF 2020 online, I also created two smart-contract challenges, `Election` and `IdleGame`. The source files of them are available on [GitHub - shw9453/my-ctf-challenges](https://github.com/shw9453/my-ctf-challenges). Here is a walkthrough of the challenge `Election`.

# Challenge

> Balsn is holding the first Shaman King election. Who will be the winner?

- Type: Smart contract
- Solves: 5/490
- Keywords: ERC223, Reentrancy, ABI encoding, Integer overflow

# Solution

## TL;DR

In this challenge, our goal is to win the election. By exploiting the `customFallback` parameter of the `_transfer` function of the extended ERC223 interface, we can reenter the `Election` contract with the auth modifier check bypassed to call the functions `propose` and `_setStage` (with some restrictions on the ABI encoding format). After proposing ourselves as a candidate, exploit the integer overflow in the `reveal` function when counting the total of votes to become the winner of the election.

## Detailed Write-up

### Overview of the Game Contract

`Election` implements a voting system using the commit-and-reveal scheme consisting of three stages: `propose`, `vote`, and `reveal`. In the `propose` stage, new candidates can be proposed by the admins (those who pass the `auth` check). In the `vote` stage, voters decide to vote for any candidate with any number of votes by submitting the hash of their ballots. The ballots should be made public in the `reveal` stage and are counted as valid if the voter has as many `ELC` tokens as the number of his total votes.

Only the admins can switch stages by calling the `_setStage` function. Besides, at the beginning of the election, two candidates are proposed and both of them have got an extremely high number of votes.

### Abusing the Custom Fallback

`Election` is also implemented as an extended ERC223 token. The extended ERC223 allows the sender to call a custom fallback function of the receiver (as long as it is a contract) during the transfer of tokens. Though the feature of custom fallback is not standardized, it exists in one of the branches of the [officially recommended implementation](https://github.com/Dexaran/ERC223-token-standard/blob/Custom-Fallback/token/ERC223/ERC223BasicToken.sol) of ERC223. The custom fallback feature was exploited in the [ATN incident](https://medium.com/coinmonks/lacking-insights-in-erc223-erc827-implementation-26be5e7c3cd7), where the attacker abused it to bypass the authorization check in the `ds-auth` library. In this challenge, we can apply the same technique to bypass the `auth` check at [line 112 to 115](https://github.com/shw9453/my-ctf-challenges/blob/master/balsn-ctf-2020/Election/public/Election.sol#L112-L115).

### Dealing with ABI Encoding

According to [line 56](https://github.com/shw9453/my-ctf-challenges/blob/master/balsn-ctf-2020/Election/public/Election.sol#L56), it seems that we are can only control the second and third parameters (type `uint` and type `bytes`, respectively) of the function we want to reenter. However, we can reenter functions with different numbers and types of parameters as long as the input data follows the ABI encoding format. [Here](https://docs.soliditylang.org/en/v0.8.1/abi-spec.html) you may find the official document of the ABI encoding.

When calling the custom fallback function, the three parameters are encoded as shown on the below left. The encoding of the parameters of the two functions to be re-entered, the `propose` function and the `_setStage` function, are shown in the middle and the right, respectively.

```
         _transfer                   propose                   _setStage 
-------------------------- -------------------------- --------------------------
|        msg.sender      | |        candidate       | |          stage         |
-------------------------- -------------------------- --------------------------
|          value         | |   offset to proposal   |
-------------------------- -------------------------- 
|     offset to data     | |     offset to name     |
-------------------------- --------------------------
|     length of data     | |   offset to policies   |
-------------------------- --------------------------
|                        | |          valid         |
|                        | --------------------------
|                        | |     length of name     |
|                        | --------------------------
|          data          | |          name          |
|                        | --------------------------
|                        | |   length of policies   |
|                        | --------------------------
|                        | |        policies        |
-------------------------- --------------------------
```

#### Reentering the Propose Function

To successfully reentry to the `propose` function, we have to craft the parameter `data` to a valid encoding of the `proposal` structure. Both the `value` and `data` parameters, including the length of `data`, are controllable by us, while the offset to `data` is always a constant of `0x60`. Notice that the offsets to the members of the `proposal` structure are all relative to the starting offset of `proposal`.

A valid exploit payload is to set the `value` to `0x40` and the length of `data` to `0xa0`, which also allows us to control `policies` at the same time (to pass the check at [line 173](https://github.com/shw9453/my-ctf-challenges/blob/master/balsn-ctf-2020/Election/public/Election.sol#L173)). To set the `value` to `0x40`, we need to own at least that number of tokens, which can be done by calling the `giveMeMoney` function, sending the minted token to another address, and repeat as many times as we want.

#### Reentering the SetStage Function

When reentering the `_setStage` function, the stage number is determined by the last byte of `msg.sender`. We need at least three accounts, whose address ends with `00`, `02` and `03` in hex, respectively, to switch to multiple stages. The `web3.privateKeyToAccount` [method](https://web3js.readthedocs.io/en/v1.2.0/web3-eth-accounts.html#privatekeytoaccount) allows us to create an account from a private key, which can be used to brute force to create accounts with a special address. The extra parameters `value` and `data` do not affect the function call.

### Winning the Most Votes

Though you may notice that the voting scheme is broken (i.e., one may vote more than the token he owns), it is still not possible to exploit only the scheme to win the election because of the expensive gas fees require to forge a large number of ballots. Simply exploiting the integer overflow at [line 145](https://github.com/shw9453/my-ctf-challenges/blob/master/balsn-ctf-2020/Election/public/Election.sol#L145) is enough for us to win `2 ** 256 - 1` votes to beat the other two candidates.

### Summary of the Exploit

1. Prepare three accounts whose address ends with `00`, `02` and `03`, respectively, using the `web3.privateKeyToAccount` method.
1. Get `0x40` tokens by exploiting the `giveMeMoney` function.
1. Vote with two ballots, one to the attacker with `2 ** 256 - 1` votes, the other to anyone with only one vote.
1. Switch to stage 0 (propose-stage) by reentering the `_setStage` function, calling from the address ending with `00`.
1. Propose the attacker as a candidate by reentering the `propose` function with valid parameters.
1. Switch to stage 2 (reveal-stage) by reentering the `_setStage` function, calling from the address ending with `02`.
1. Reveal the ballots submitted in Step 3.
1. Switch to stage 3 (end-stage) by reentering the `_setStage` function, calling from the address ending with `03`.
1. Call `giveMeFlag` from the attacker to set the `sendFlag` variable to `true`.

Flag: `BALSN{M4k3_Reen7r4ncy_gre47_ag41n}`
