---
title: "Real World CTF Finals 2019"
categories:
  - blog
tags:
  - ctf
  - smart contract
  - crypto
---

Glad to participate in the on-site finals of [Real World CTF](https://ctftime.org/event/857) in Beijing! Here I would like to share the solution to the challenges I solved.

# Bank2

- Type: Crypto
- Keywords: Elliptic curve, Schnorr signature algorithm

## Description

> Our bank has invested in a HUGE security upgrade. Now we are equipped with the latest interactive multi-signature protocol to keep your assets safe. Your satisfaction is our first priority.

## Solution

Similar to the challenge `Bank` in Real World CTF 2019 Quals ([Ref](https://ctftime.org/task/9225)), the server-side implements the Schnorr signature algorithm. The difference between these two implementations is the verifying function:

```python
def cosi_verify(c, s, pk, m):
    if (not on_curve(pk)):
        print('Not on curve')
        return False
    cPrime = sha256(bytes_point(point_add(point_mul(G, s), point_mul(pk, n-c))) + m)
    if cPrime == c:
        return True
    return False
```

The main logic of the server's code is:

```python
# generate server's public-private key pair
sk, pk = generate_keys()
print('sk, pk =', sk, pk)
...
if msg[0] == '0': # withdraw
    ...
    if cosi_verify(C, S, pk, 'WITHDRAW'):
        req.sendall("Here is your coin: %s\n" % FLAG)
if msg[0] == '1': # deposit
    r = p - Random.random.randint(5, p/2**16)
    req.sendall("%s\n" % repr((point_mul(G, r), pk)))
    ...
    c = sha256(bytes_point(T) + 'DEPOSIT')
    s = r + c * sk
    req.sendall("%s\n" % repr(s))
    ...
    if cosi_verify(C, S, PK, 'DEPOSIT'):
        balance += 100
        req.sendall('Coin Deposited')
```

In the deposit method, `T` and `PK` are provided by us. The bug happens at line 14, it should be `s = (r + c * sk) % n` instead (credit to [@fweasd](https://github.com/b04902036)). Notice that the `s` we received equals to `p - r' + c * sk`, where `r'` is around 240 bits. Since we know the value of `p`, `c`, we can calculate the value of `sk = (s - p) // c + 1`, which is the secret key of the server. Then, we sign the message `"WITHDRAW"` with the secret key and get the flag by the withdraw method.

Exploit:

```python
G = ... # base point
n = ... # order of the group
p = ... # mod p

def sign(sk, msg):
    rn = 0x9453
    T = point_mul(G, rn)
    c = sha256(bytes_point(T) + msg)
    s = (rn + c * sk) % n
    return c, s, T

r = remote(host, port)
sk, pk = generate_keys()
c, s, T = sign(sk, 'DEPOSIT')
server_s = deposit(r, (T, pk), (c, s))

server_sk = (server_s - p) // c + 1
c, s, _ = sign(server_sk, 'WITHDRAW')
withdraw(r, (c, s))

# rwctf{2r0unD_Schn0Rr_1s_N07_5AfE._cHEck_tH3_0ak1aNDl9_p4p3r}
```

## Montagy

- Type: Smart contract
- Keywords: Solidity inline assembly, EVM bytecode, z3

## Solution

### TL;DR

By the `newPuzzle` method, create a new puzzle that has the same tag value as one of the existing puzzles to pass the `isOfficialChecksum` check. Let our new puzzle call `solve` to the proxy contract, and the proxy contract will send out all the ether it has.

### Detailed Write-up

In this challenge, our goal is to let the balance of the proxy contract become 0, and then we will receive the flag in a transaction sent from the game server.

The code of the proxy contract is:

```js
pragma solidity ^0.5.11;

contract Montagy{
    address payable public owner;
    mapping(bytes32=>bool) isOfficialChecksum;
    ...
    
    modifier onlyPuzzle(){
        require(puzzleChecksum[msg.sender] != 0);
        _;
    }
    ...
    
    function registerCode(bytes memory a) public onlyOwner {
        isOfficialChecksum[tag(a)]=true;
    }
    
    function newPuzzle(bytes memory code) public onlyGameOn returns(address addr){
        bytes32 cs = tag(code);
        require(isOfficialChecksum[cs]);
        
        addr = deploy(code);
        lastchildaddr=addr;
        puzzleChecksum[addr] = cs;
    }
    
    function solve(string memory info) public onlyGameOn onlyPuzzle {
        owner.transfer(address(this).balance);
        winnerinfo = info;
    }
    ...
    
    function tag(bytes memory a) pure public returns(bytes32 cs){
        assembly{
            let groupsize := 16
            let head := add(a,groupsize)
            let tail := add(head, mload(a))
            let t1 := 0x21711730
            let t2 := 0x7312f103
            let m1,m2,m3,m4,p1,p2,p3,s,tmp
            for { let i := head } lt(i, tail) { i := add(i, groupsize) } {
                s := 0x6644498b
                tmp := mload(i)
                m1 := and(tmp,0xffffffff)
                m2 := and(shr(0x20,tmp),0xffffffff)
                m3 := and(shr(0x40,tmp),0xffffffff)
                m4 := and(shr(0x60,tmp),0xffffffff)
                for { let j := 0 } lt(j, 0x4) { j := add(j, 1) } {
                    s := and(add(s, 0x68696e74),0xffffffff)
                    p1 := sub(mul(t1, 0x10), m1)
                    p2 := add(t1, s)
                    p3 := add(div(t1,0x20), m2)
                    t2 := and(add(t2, xor(p1,xor(p2,p3))), 0xffffffff)
                    p1 := add(mul(t2, 0x10), m3)
                    p2 := add(t2, s)
                    p3 := sub(div(t2,0x20), m4)
                    t1 := and(add(t1, xor(p1,xor(p2,p3))), 0xffffffff)
                }
            }
            cs := xor(mul(t1,0x100000000),t2)
        }
    }
}
```

The full source code can be found on Ropsten: [Proxy contract](https://ropsten.etherscan.io/address/0xd068fcc44525569fb593189c8f22827cf0f50f3f#code)

By inspecting the transactions to the proxy contract, we may notice that the owner had registered and deployed two puzzles via the `registerCode` and `newPuzzle` methods. Also, their source code can be found on Ropsten: [side contract 1](https://ropsten.etherscan.io/address/0xe45d9f944fe4aa364b7ea958f694fb59af8cdee6#code), [side contract 2](https://ropsten.etherscan.io/address/0x6e6537d0fd1f12ccbcdbb5a9e832885362239aa9#code). However, these two puzzles are not computationally feasible to solve.

How about creating a new puzzle that calls `server.solve()` directly? To make this happen, the new puzzle should be deployed via `newPuzzle` and should pass the `isOfficialChecksum` check at line 20. While only the owner is allowed to register a new checksum (tag value), we have to construct a new puzzle that has the same tag value as one of the existing puzzles.

First, write a contract which simply calls `solve` to the proxy contract:

```js
contract P3 {
    Montagy public server;
    constructor() public {
        server = Montagy(0xD068fcC44525569fB593189c8f22827cF0f50f3f);
    }
    function do_solve() public {
        server.solve('balsn');
    }
}
```

Compile this contract and get its deploy bytecode. Note that whatever we pad at the end of the bytecode will not change the deploy result. Therefore, we choose to find proper padding to our bytecode to control its tag value. According to the source code of the proxy contract, to calculate the tag value, the method `tag` iterates from the start of the bytecode to the end. Since we can calculate the tag value of the unpadded bytecode beforehand, we only have to focus on the last iteration, where the padding is the input bytes.

Here is a script for finding proper padding:

```python
from z3 import *

def find(last, target):
    t1, t2 = int(last[:8], 16), int(last[8:], 16)
    tar1, tar2 = int(target[:8], 16), int(target[8:], 16)

    s = 0x6644498b
    s = BitVecVal(s, 256)
    m1 = BitVec('m1', 256)
    m2 = BitVec('m2', 256)
    m3 = BitVec('m3', 256)
    m4 = BitVec('m4', 256)

    for j in range(4):
        s = (s + 0x68696e74) & 0xffffffff
        p1 = (t1<<4) - m1
        p2 = t1 + s
        p3 = (t1>>5) + m2
        t2 = (t2 + (p1^(p2^p3))) & 0xffffffff
        p1 = (t2<<4) + m3
        p2 = t2 + s
        p3 = (t2>>5) - m4
        t1 = (t1 + (p1^(p2^p3))) & 0xffffffff

    sol = Solver()
    sol.add(And(t1 == tar1, t2 == tar2))
    if sol.check():
        m = sol.model()
        m_l = map(lambda x: m[x].as_long(), [m4, m3, m2, m1])
        pad = 0
        for x in m_l:
            pad <<= 0x20
            pad |= x
        return hex(pad)[2:].zfill(32)
    else:
        raise Exception('No solution')
```

Append the padding to our deploy bytecode, and call `newPuzzle` in the proxy contract with the padded bytecode as the parameter. After our contract is deployed, call the `do_solve` method in our contract. All ether that the proxy contract has will be sent out, and then we will get the flag: `rwctf{cOd3_i5_CH3ep_sHoW_mE_tHe_OPcdE...Oh_&&_1_CUP_of_T_PlEA53_:P}`
