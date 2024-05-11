---
title: EVM - Init and Runtime code
date: 2024-04-14
categories: [solidity,evm]
tags: [evm,solidity]    
description: In-depth explanation.
pin: true
---

Also posted at <https://ethereum.stackexchange.com/questions/76334/what-is-the-difference-between-bytecode-init-code-deployed-bytecode-creation/163307#163307>

Let's start by deploying a simple contract and see what's happening at the low level.

```
pragma solidity ^0.8.3;

contract Counter {
    constructor() {}
    
    function add() public pure returns(uint) {
        return 5+5;
    }

}
``` 
In OPCODE Output:

```
[00]	PUSH1	80
[02]	PUSH1	40
[04]	MSTORE	
[05]	CALLVALUE	
[06]	DUP1	
[07]	ISZERO	
[08]	PUSH2	0010
[0b]	JUMPI	
[0c]	PUSH1	00
[0e]	DUP1	
[0f]	REVERT	
[10]	JUMPDEST	
[11]	POP	
[12]	PUSH1	b6
[14]	DUP1	
[15]	PUSH2	001f
[18]	PUSH1	00
[1a]	CODECOPY	
[1b]	PUSH1	00
[1d]	RETURN	
[1e]	INVALID	
[1f]	PUSH1	80
[21]	PUSH1	40
[23]	MSTORE	
[24]	CALLVALUE	
[25]	DUP1	
[26]	ISZERO	
[27]	PUSH1	0f
[29]	JUMPI	
[2a]	PUSH1	00
[2c]	DUP1	
[2d]	REVERT	
[2e]	JUMPDEST	
[2f]	POP	
[30]	PUSH1	04
[32]	CALLDATASIZE	
[33]	LT	
[34]	PUSH1	28
[36]	JUMPI	
[37]	PUSH1	00
[39]	CALLDATALOAD	
[3a]	PUSH1	e0
[3c]	SHR	
[3d]	DUP1	
[3e]	PUSH4	4f2be91f
[43]	EQ	
[44]	PUSH1	2d
[46]	JUMPI	
[47]	JUMPDEST	
[48]	PUSH1	00
[4a]	DUP1	
[4b]	REVERT	
[4c]	JUMPDEST	
[4d]	PUSH1	33
[4f]	PUSH1	47
[51]	JUMP	
[52]	JUMPDEST	
[53]	PUSH1	40
[55]	MLOAD	
[56]	PUSH1	3e
[58]	SWAP2	
[59]	SWAP1	
[5a]	PUSH1	67
[5c]	JUMP	
[5d]	JUMPDEST	
[5e]	PUSH1	40
[60]	MLOAD	
[61]	DUP1	
[62]	SWAP2	
[63]	SUB	
[64]	SWAP1	
[65]	RETURN	
[66]	JUMPDEST	
[67]	PUSH1	00
[69]	PUSH1	0a
[6b]	SWAP1	
[6c]	POP	
[6d]	SWAP1	
[6e]	JUMP	
[6f]	JUMPDEST	
[70]	PUSH1	00
[72]	DUP2	
[73]	SWAP1	
[74]	POP	
[75]	SWAP2	
[76]	SWAP1	
[77]	POP	
[78]	JUMP	
[79]	JUMPDEST	
[7a]	PUSH1	61
[7c]	DUP2	
[7d]	PUSH1	50
[7f]	JUMP	
[80]	JUMPDEST	
[81]	DUP3	
[82]	MSTORE	
[83]	POP	
[84]	POP	
[85]	JUMP	
[86]	JUMPDEST	
[87]	PUSH1	00
[89]	PUSH1	20
[8b]	DUP3	
[8c]	ADD	
[8d]	SWAP1	
[8e]	POP	
[8f]	PUSH1	7a
[91]	PUSH1	00
[93]	DUP4	
[94]	ADD	
[95]	DUP5	
[96]	PUSH1	5a
[98]	JUMP	
[99]	JUMPDEST	
[9a]	SWAP3	
[9b]	SWAP2	
[9c]	POP	
[9d]	POP	
[9e]	JUMP	
[9f]	INVALID	
[a0]	LOG2	
[a1]	PUSH5	6970667358
[a7]	INVALID	
[a8]	SLT	
[a9]	KECCAK256	
[aa]	GASLIMIT	
[ab]	INVALID	
[ac]	INVALID	
[ad]	INVALID	
[ae]	INVALID	
[af]	INVALID	
[b0]	RETURNDATASIZE	
[b1]	CREATE	
[b2]	RETURNDATACOPY	
[b3]	INVALID	
[b4]	INVALID	
[b5]	SHL	
[b6]	PUSH17	91c4be8806046d5260161484ea4b22089a
[c8]	INVALID	
[c9]	INVALID	
[ca]	PUSH5	736f6c6343
[d0]	STOP	
[d1]	ADDMOD	
[d2]	ISZERO	
[d3]	STOP	
[d4]	CALLER
```
Let's start from the first line ```PUSH1 80; PUSH1 40; MSTORE``` , now what's happening here is the value 0x80 is stored in 0x40 which is just the initialization of the free memory pointer; and now what does that mean? 

- Free memory pointer points to a place where we can store whatever we want in memory right now meaning; we can store something right there in that location without worrying if that place is used by something else.

From the post https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html

- Solidity reserves four 32-byte slots, with specific byte ranges (inclusive of endpoints) being used as follows:

> 0x00 - 0x3f (64 bytes): scratch space for hashing methods

> 0x40 - 0x5f (32 bytes): currently allocated memory size (aka. free memory pointer)

> 0x60 - 0x7f (32 bytes): zero slot

0x7f is 127 in decimal and 128 is 0x80

- As we can see that till 0x7f it's being used up for certain stuffs reserved by the solidity compiler and the next memory location which is 0x80 is from where we can store the stuffs we want. So 0x80 is our free location where we can store anything! Now where do we store this location? Yes! In 0x40 which is the place reserved to store the free memory location.

---

Now let's go to the next lines;

```
[05]    CALLVALUE   
[06]    DUP1    
[07]    ISZERO  
[08]    PUSH2   0010
[0b]    JUMPI   
[0c]    PUSH1   00
[0e]    DUP1    
[0f]    REVERT  
```
The call value opcode returns the amount sent with the transaction, currently our constructor is not marked as payable; meaning we can't send an amount when deploying the contract.

> These lines are checking if we sent some amount to the contract when deploying it.

Now say we sent some amount when creating the contract so the CALLVALUE opcode returns this value and the ISZERO opcode returns 0; meaning some value is sent with the transaction. If there was no amount sent with the transaction then CALLVALUE returns 0 and ISZERO returns 1, saying no value was sent with the transaction. 

```PUSH2 0010; JUMPI``` , The JUMPI instruction jumps to the location 0010  if the value on top of the stack is 1(that is true), so now in our contract say we sent some value with the transaction and ISZERO set the top to 0(false) so this jump won't happen and the execution continues till the next lines.

```
[0c]    PUSH1   00
[0e]    DUP1    
[0f]    REVERT  
```
> The contract reverts. This is what happens at the low-level if you sent sent some amount when the constructor is marked as non payable.

---

Now the scenario where we didn't send any amount when deploying the contract , which is exactly what we want since we didn't mark the contructor() as payable.

- The JUMPI instruction jumps to 0010 where the next set of lines start.

```
[10]	JUMPDEST	
[11]	POP	
[12]	PUSH1	b6
[14]	DUP1	
[15]	PUSH2	001f
[18]	PUSH1	00
[1a]	CODECOPY	
```

> These lines are the most important part and this where you can understand the difference between init and runtime code. You can get a practical overview of this if you head to evm.codes and run all these instructions by yourself.

- So right now this entire code is the initialization code and from 001f is where the actual runtime code starts and b6 is the size of the runtime code.

The COPDECOPY opcode copies b6 amount of size from [1f] and returns it. And this whole includes our runtime code which is the code that runs when we interact with the contract. 

From the word itself initialization code, it's obvious that it's run when deploying the contract from where the actual runtime code is being copied to evm.

> You'll have a much more clear understanding of this when we do this on our own without the CODECOPY opcode.

If you head over to [1f]:

```
[1f]	PUSH1	80
[21]	PUSH1	40
[23]	MSTORE	
[24]	CALLVALUE	
[25]	DUP1	
[26]	ISZERO	
[27]	PUSH1	0f
[29]	JUMPI	
[2a]	PUSH1	00
[2c]	DUP1	
[2d]	REVERT	
[2e]	JUMPDEST	
[2f]	POP	
[30]	PUSH1	04
[32]	CALLDATASIZE	
[33]	LT	
[34]	PUSH1	28
[36]	JUMPI	
[37]	PUSH1	00
[39]	CALLDATALOAD	
```

This looks similar to the one when we deployed the contract right? Yes!

This is the code that runs on the blockchain when you make any kind of interaction with this smart contract , meaning anything like if you want call a function or just read something with a function this is the code that executes.

- But how does the contract know which function we called ?, yes the CALLDATALOAD opcode gives us the output , as per the abi encoding specification the function name and parameters are encoded using Keccak hash and sent with the transaction as calldata. 
- So I think now you can see where this is going right? We check the desired function using calldataload and jumps to that particular location. 
- If we call the add() function meaning we sent a transaction to this contract with calldata as '4f2be91f' , the evm checks the output of calldataload and jumps to the location of the add function and from where which the addition operation takes place. 
- If we had another function say sub() the evm jumps us to the location of the sub() function.

---

Now let's do all of this by our own. 

Let's create a simple contract that add two numbers but guess what we're going to build this one by our hand; Yes! by writing pure raw opcodes.

- First let's write the code that actually runs on the blockchain when you interact with it aka runtime code.

```
PUSH1	04
PUSH1	03
ADD	
PUSH1	00
MSTORE	
PUSH1	20
PUSH1	00
RETURN	
``` 

- We're pushing 4 and 3 to the stack and the add opcode adds these two numbers and stores it in the stack, so 7 is now on top of the stack.

- The MSTORE opcode copies 7 to the memory location 0 , which is the first memory location.

- The return opcodes returns 0x20 bytes from the offset 0 , meaning it gives us 32 bytes from the 0 offset that is we get 0...7 as the return value.
> So if we make any kinds of transaction to this contract , like absolutely anything it returns 7. 

If we call random(), meow(), cat() or anything it just returns 7.

---

But how do we deploy this contract to the blockchain, can we do this directly? Nope.

Whenever we deploy our code to the blockchain who is the one that is responsible for copying this code to the evm's memory? 

Say the contract had a constructor that accepts and changes the value of certain variables when deploying the contract, how does this work? An NFT is given different names when the constructor is called by different users.

So we need something does all of these for us. Yes! The initialization code is responsible for this.

```
PUSH13	600460030160005260206000f3
PUSH1	00
MSTORE	
PUSH1	0d
PUSH1	13
RETURN	
```

This is our initialization code for this contract.

Our add contract in opcodes when converted to bytecode is `600460030160005260206000f3`, which is 13 bytes long.

- All this does is stores our runtime code in memory location 0 and returns this value. Which is exactly what the evm wants. 
- This init code just runs first time when you deploy the contract, all it does is just copies the runtime code which is the actual code of the contract to the evm's memory. The same thing happens with the codecopy opcode.

When converted to bytecode it looks like this ```0x6c600460030160005260206000f3600052600d6013f3``` 
Now you can see why the init code is long compared to the runtime code, because the init code includes the runtime code also. Check an example contract in etherscan and you'll see.

---

Now let's deploy our contract!

```
cast send  --private-key $PKEY --rpc-url $RPCK --create 0x6c600460030160005260206000f3600052600d6013f3
```

Now I'm going to deploy this contract to sepolia network.

https://sepolia.etherscan.io/address/0xd10422428c9C162b403E59d223D74A2C88fe0083#code

Check the code section and click opcodes view and you can see the runtime code!

Now let's interact with it.

```
cast call --rpc-url $RPCK 0xd10422428c9C162b403E59d223D74A2C88fe0083 "IDontExistFunction()"
```

What do you think is going to be returned? Yes! 7!

```
0x0000000000000000000000000000000000000000000000000000000000000007
```

---







