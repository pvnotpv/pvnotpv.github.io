---
title: Ethernaut - GakeKeeperOne walkthrough (Finding gas using the debug method)
date: 2024-04-22
categories: [solidity]
tags: [solidity,ethernaut]     
description: Solution to this brainfuck level
---

- This level has taught me enough pain and suffering, took almost half a day to solve and my eyes were completely dead.

![dead](/images/day21.png)

The contract: 

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

```
modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }
```

- This modifier just means the one calling the contract should be another contract and not the user itself.

```
modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }
```

- Solving this part was also not so hard and I've written a simple contract below as a POC

```
import "forge-std/console.sol";

contract POC {

    function tester(bytes8 key, address _addr) public {

        console.log(uint32(uint64(key)));
        console.log(uint16(uint64(key)));

        console.log(uint64(key));

        console.log(tx.origin);

        console.log(uint16(uint160(_addr)));

    }
}
```

So the bytes require an bytes8 value, the max 8 bytes number is 0xFFFFFFFFFFFFFFFF

Each character in hex represent 4 bits , so 2 digits in hex is a single byte.

# Type casting explanation (using chisel)

```
➜ bytes8 b8 = bytes8(0xFFFFAFFFFF3FFFFF)
➜ b8
Type: bytes8
└ Data: 0xffffafffff3fffff000000000000000000000000000000000000000000000000
➜
```
Now let's convert that to bytes4 and see what happens.

```
➜ bytes4 b4 = bytes4(b8)
➜ b4
Type: bytes4
└ Data: 0xffffafff00000000000000000000000000000000000000000000000000000000
➜
```

As we can see that half the part in the left is kept but the right part has been just gone.

So this is in the case of bytes, now let's check the case with integers 

```
➜ uint64 u64 = uint64(b8)
➜ u64
Type: uint64
├ Hex: 0xffffafffff3fffff
├ Hex (full word): 0x000000000000000000000000000000000000000000000000ffffafffff3fffff
└ Decimal: 18446656112766746623
➜
```
Now this makes sense because 8 bytes is 64 bits so there would be no information loss.

```
➜ uint32 u32 = uint32(u64)
➜ u32
Type: uint32
├ Hex: 0xff3fffff
├ Hex (full word): 0x00000000000000000000000000000000000000000000000000000000ff3fffff
└ Decimal: 4282384383
➜
```
- In the case of integers we can see that the left part is being removed insted of what happens in the case of bytes where the right part was being removed.

```
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
```
So we first we can provide with a value so that even when converted to uint16 , it should equal the one with uint32; Now this isn't a huge deal because from earlier we saw that the right part is the one that's being removed so we can provide it with a value so that the end will stay even when convert to uint16.

Hers's one example:- 0x0000000000000111

This number has no information loss when convert to uint16, so that's our first part , the next is it should't equal to the one with uin64. For that we can just add a random character to the left side.

So it becomes:- 0x0300000000000111

Now our second part is also solved.

Now the last part.

Let's see what happens if we run it with an address.

uint16(uint160(0xe24d5514FEAFd1985d4e473B8e73E90EcdC103cc)), it gives the output as 972.

So all we have to do is to change the right side of the hex from earlier to the hex value of 972 which is 0x3cc, so the key becomes:

0x03000000000003cc , which is our final key.

--- 

Now the most important part is solving the gas part which requires debugging so I can't show it here but I'll provide the hints.

So let's multiply 8191 with 3 so that the number gives us zero reminder after a modulus operation which is 24537

So this much gas should available after running gasleft operation , which is the value that is going to be pushed down the stack. 
The most important part here is that the gas opeartion also takes 2 gas, so when providing with the payload it should also be considered.

- Also you should enable optimization to 1000 and compile the contract with 0.8.12 compiler to the correct gas because , optimization and different compiler changes the number of opcodes which is going to change the gas count entirely.

Here's the final poc to solve the level.

Remember you should compile with these options because you check the level's contract in etherscan and yea...

<#lang=en&optimize=true&runs=1000&evmVersion=null&version=soljson-v0.8.12+commit.f00d7308.js>

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface GatekeeperOne {
    function enter(bytes8 _gateKey) external payable returns (bool);
}

contract Meh {
    
    GatekeeperOne kone;

    constructor(address _addr) payable {
        kone = GatekeeperOne(_addr);
    }

    function smendEther(uint _gas) public payable {
        kone.enter{gas: _gas}(0x03000000000003cc);
    }

    receive() external payable { }

    fallback() external payable { }


}
```
- So the level was a real challenge but I've learned hella lot of stuffs about debugging , gas and solidity in general.
