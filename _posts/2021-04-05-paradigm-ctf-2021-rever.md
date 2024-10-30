---
title: "Paradigm CTF 2021 - Rever"
categories:
  - blog
tags:
  - ctf
  - smart contract
---

In Feb. this year, I and some other Balsn members participated in [Paradigm CTF](https://ctf.paradigm.xyz/), an Ethereum-focused security competition held by [Paradigm](https://twitter.com/paradigm). First of all, thanks to the organizers for preparing such high-quality and impressive challenges! Luckily, our team, `whoami`, got fourth place with 11 out of 17 challenges solved.

![pctf-solve.png](https://shw9453.github.io/assets/images/pctf-solve.png)

When browsing the [official repository](https://github.com/paradigm-operations/paradigm-ctf-2021), I noticed a clever trick used in a coding challenge, `REVER`, and wondered whether I could improve the solution furthermore. Here, I will share the optimal solution I can develop and some tips for EVM bytecode golfing.

# Challenge

> Writing contracts that can only be used in one direction is so last year.

- Type: Smart contract
- Solves: 5/67
- Keywords: Yul, EVM opcode, EVM bytecode, code golfing
- [Source files](https://github.com/paradigm-operations/paradigm-ctf-2021/tree/master/rever/public)

# Solution

## Overview of the Challenge

The goal of this challenge is to write a contract (i.e., a sequence of EVM bytecodes) that can [identify](https://github.com/paradigm-operations/paradigm-ctf-2021/blob/master/rever/public/contracts/Setup.sol#L70-L83) palindromes. Even more complicated, our written contract should work as well when executed in the reverse direction (somehow inspired by the movie `TENET`).

Both original and reversed versions of our contract are deployed after passing [checks](https://github.com/paradigm-operations/paradigm-ctf-2021/blob/master/rever/public/contracts/Setup.sol#L47-L48) on the length and the included opcodes. When we request a flag, the backend judges the two contracts with fixed and randomly generated [test cases](https://github.com/paradigm-operations/paradigm-ctf-2021/blob/master/rever/public/deploy/chal.py#L10-L42). Our contract should answer all test cases correctly to get a flag.

## The First Step

By a careful inspection, we may realize that this challenge is a **real** coding challenge since the possible ways to call external contracts are forbidden. Besides, our contract's length is limited to a maximum of 100 bytes.

How can we make our contract valid when executed in the reverse direction? A simple way is to palindrome it (i.e., `s' = s[:-1] + s[::-1]`). As a result, only half of our contract is executed in both directions. From now on, we only need to focus only on the first half of our contract, whose length is at most 50 bytes long.

First, let's take a look at the [official solution](https://github.com/paradigm-operations/paradigm-ctf-2021/blob/master/rever/private/impl.yul), which is the compiled bytecode (with the last 3 bytes discarded) from the following code written in [Yul](https://docs.soliditylang.org/en/v0.8.3/yul.html):

```javascript
{           
    let i := returndatasize()
    let m := shr(selfbalance(), calldatasize())
    let e := sub(calldatasize(), selfbalance())
    for {} and(
        eq(shr(248, calldataload(i)), shr(248, calldataload(sub(e, i)))),
        lt(i, m)
    ) {i := add(i, selfbalance())} {}
    
    mstore(callvalue(), eq(i, m))
    return(callvalue(), 0x20)
}
```

This code implements a straight-forward byte-by-byte comparison algorithm. Notice that after deploying the two contracts, `1 wei` is [transferred](https://github.com/paradigm-operations/paradigm-ctf-2021/blob/master/rever/private/Exploit.sol#L8-L9) to both of them to let the return value of `selfbalance()` become 1. As for `callvalue()` and `returndatasize()`, they are always 0 afterwards.

## Optimizing the Solution

Although the above implementation is good enough to solve this challenge, one may find space to improve by inspecting the EVM bytecode directly (e.g., redundant opcodes, stack elements re-arrangement). Let me show you the optimal solution I can figure out, which is 24 bytes long:

```
pc    instruction       stack (top -> bottom)
----  --------------    ----------------------------------
0000  CALLDATASIZE      [size]
0001  RETURNDATASIZE    [0 size]

0002  JUMPDEST          [i size]
0003  DUP1              [i i size]
0004  SELFBALANCE       [1 i i size]
0005  ADD               [i+1 i size]
0006  SWAP1             [i i+1 size]

0007  CALLDATALOAD      [data[i] i+1 size]
0008  DUP2              [i+1 data[i] i+1 size]
0009  DUP4              [size i+1 data[i] i+1 size]
000A  SUB               [size-i-1 data[i] i+1 size]
000B  CALLDATALOAD      [data[size-i-1] data[i] i+1 size]
000C  XOR               [xor i+1 size]
000D  RETURNDATASIZE    [0 xor i+1 size]
000E  BYTE              [byte i+1 size]
000F  RETURNDATASIZE    [0 byte i+1 size]
0010  JUMPI             [i+1 size]

0011  DUP2              [size i+1 size]
0012  DUP2              [i+1 size i+1 size]
0013  LT                [i+1<size i+1 size]
0014  PUSH1 0x02        [0x02 cont i+1 size]
0016  JUMPI             [i+1 size]
0017  STOP              [i+1 size]
```

This piece of bytecode consists of 4 parts:

1. Initialize `i` and `size`.
1. Increase `i` at the beginning of each loop.
1. Compare `s[i]` and `s[size-1-i]`. If they are the same, continue. If not, jump to offset 0 to revert the transaction.
1. Jump to the beginning of the loop if `i + 1 < size`; otherwise, stop the execution.

For reference, below is the pseudocode with the same behavior as the bytecode.

```javascript
{
    let size := calldatasize()
    let i := returndatasize()
    do {
        let j := add(i, selfbalance())
        if byte(
            returndatasize(),
            xor(calldataload(i), calldataload(sub(size, j)))
        ) { jump(returndatasize()) }
        i := j
    }
    while lt(j, size)
    stop()
}
```

## Golfing in EVM Bytecode

**Tip #1: Replace constants with environmental and block variables**

This trick is applied to optimize the use of constants 0 and 1 in the official solution. As shown above, `returndatasize()` and `callvalue()` replace the constant 0, while `selfbalance()` replace the constant 1. By this method, the use of 0 and 1 reaches the optimal of 1 byte for each use.

The reason for choosing `returndatasize()` and `selfbalance()` to replace the constants is that the former is always 0 (before any function calls), and the latter is usually controllable by us. Remember that the balance scale is calculated in `wei`. Other variables such as `chainid()` or `number()` might also be exploited but are less useful in general.

**Tip #2: Push once, duplicate multiple times**

For other frequently-used constants that cannot be presented by an environmental or block variable, push them onto the stack (e.g., `PUSH2 0x9453`) then duplicate them (e.g., `DUP1`) whenever using them to save (at least) 1 byte for every additional use.

**Tip #3: Take advantage of special opcodes**

Use powerful opcodes such as `ADDMOD`, `MULMOD`, `BYTE` to simplify arithmetic operations. For example, to get the first byte of `x`, use `byte(0, x)` instead of `shr(248, x)` to save 1 byte. If there is no need to change the input data, consider pushing it onto the stack by `calldataload` instead of `calldatacopy` to the memory and then `mload` to the stack.

**Tip #4: Avoid POP by reusing the old copy of variables**

We choose to increase `i` by one (part 2) **before** indexing the input data (part 3) to avoid wasting the old copy. In normal cases, the compiler tends to pop out the old copy of a variable after updating it:

```
0010  DUP1              [i i size]
0011  CALLDATALOAD      [data[i] i size]
...
0020  DUP1              [i i size]
0021  SELFBALANCE       [1 i i size]
0022  ADD               [i+1 i size]
0023  SWAP1             [i i+1 size]
0024  POP               [i+1 size]
```

However, making use of the old copy would let us save a `POP` and a `DUP`:

```
0010  DUP1              [i i size]
0011  SELFBALANCE       [1 i i size]
0012  ADD               [i+1 i size]
0013  SWAP1             [i i+1 size]
0014  CALLDATALOAD      [data[i] i+1 size]
...
```

**Tip #5: Exploit execution error to revert the transaction**

Instead of explicitly using the `REVERT` opcode, an execution error in EVM has the same effect of reverting without any data returned. The most common way is by the byte `fe`, also called the `INVALID` instruction, which is only 1 byte long. If you want to revert based on a particular condition, use `JUMPI` to jump to an offset that is not a `JUMPDEST` to revert. See the 3rd part of the above bytecode as an example.

# Misc

Actually, I didn't come up with the above solution during the competition. I used a more complicated way to calculate the numerical difference between the first and second half of the input string, which even has a certain chance of failure.

By the way, there seems to be an easy unintended solution: return a hard-coded answer for the five fixed test cases (determined by the length of the input string), and guess for the randomly generated ones. The probability of success is to be `2^-10`, which is feasible to brute-force until success within only hours.
