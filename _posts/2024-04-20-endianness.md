---
title: Solidity - Deep dive into Endianness and Bytes 
date: 2024-04-16 
categories: [solidity]
tags: [solidity]     
description: In depth-overview
---

Continuation of - <https://pvnotpv.github.io/posts/solslayout/>

Different languages read their text in different orders. English reads from left to right, for example, while Arabic is read right to left.
If my computer reads bytes from left to right, and your computer reads from right to left, we're going to have issues when we need to communicate.

Endianness is represented two ways Big-endian (BE) and Little-endian (LE).

- BE stores the big-end first. When reading multiple bytes the first byte (or the lowest memory address) is the biggest - so it makes the most sense to people who read left to right.
- LE stores the little-end first. When reading multiple bytes the first byte (or the lowest memory address) is the littlest -  so it makes most sense to people who read right to left.

> The stack grows from higher to lower memory address btw.

If I took the decimal number 2,984, what number could you change to change the number by the smallest amount? The 4. If I change the 4 to 5, the whole number only goes up by 1.
But let's say you change the 2 in 2,984. It will change the number significantly and go up by a thousand.
This is the exact same with bytes and bits.
We refer to the byte holding the smallest position as the Least Significant Byte (LSbyte) and the bit holding the smallest position as the Least Significant Bit (LSbit).

The number 348 in binary is 101011100 , which is the least significant byte ? 
The zero in the right end. If we add 1 to the number it becomes 101011101 , which is 349

101011101 - 349
101011110 - 350
101011111 - 351

Now what if I change the bit in the left most one 

001011111 - 94
111011110 - 478 

So the left one is the most significant bit , because it makes a huge difference. 
The right one is the least significant becausae it makes the least difference.

---

0x12345678

In little-endian, it would be stored as:

Address    Data
1000       78   <-- lowest memory address stores the least significant byte
1001       56
1002       34
1003       12

In big-endian, it would be stored as:

Address    Data
1000       12   <-- lowest memory address stores the highest significant byte
1001       34
1002       56
1003       78

---

In solidity

Big endian format : strings and bytes
Little endian format : other types (numbers, addresses, etc…).

```
pragma solidity 0.8;

contract StorageLayout {
    uint256 x;      <--- Slot0
    string str;     <--- Slot1
    uint256 y;      <--- Slot2
    address addr;   <--- Slot3
    bytes32 _byte;  <--- Slot4

    function setVar(uint256 a, string memory b, uint256 c, address _addr, bytes32 byte_) public {
        x = a;
        str = b;
        y = c;
        addr = _addr;
        _byte = byte_;
    }

    // Below function is from Jesper Kristensen's video

    function readStorageSlot(uint256 i) public view returns (bytes32 content) {
        assembly {
            content := sload(i) // just a low level function to get the value stored in slots. if i=1 then it outputs the value stored in slot 1.
        }
    }
}

```

Let the call the function with 

```
x = 23
str = 'poda'
y = 67
addr = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4
_byte = 0x6162636400000000000000000000000000000000000000000000000000000000

```
Now let's check the slot


Slot0 which is x     -> 0x0000000000000000000000000000000000000000000000000000000000000017

Slot1 which is str   -> 0x706f646100000000000000000000000000000000000000000000000000000008

Slot2 which is y     -> 0x0000000000000000000000000000000000000000000000000000000000000043

Slot3 which is addr  -> 0x0000000000000000000000005b38da6a701c568545dcfcb03fcb875f56beddc4

Slot4 which is _byte -> 0x6162636400000000000000000000000000000000000000000000000000000000

Big endian    = Strings and Bytes
Little endian = Addresses and Integers

Let's take this example 

```
Slot0 which is x     -> 0x0000000000000000000000000000000000000000000000000000000000000017

```

23 in hex is 0x17

24 in hex is 0x18

So which is the least significant here ?, 7 right.

---

```
Slot1 which is str   -> 0x706f646100000000000000000000000000000000000000000000000000000008
```

'poda' in hex 70 6F 64 61 
---

### Bytes

```
pragma solidity ^0.8.0; 
  
contract BytesContract {

    bytes1 n;
    function setVar(bytes1 a) public returns (bytes1) {
        n = a;
        return n;
    }

}
```
bytes1 means the max value the 'n' variable can hold is a single byte and how much is a single byte?, 8 bits or the max value is 11111111, which is 255 in decimal and 0xFF in hex.

so 0xFF is the max value this variable can hold.

What do you think is gonna happen if we provide 256 to this function?, yep it throws an error because we cannot represent the number within a single byte

```
transact to FixedSizeBytesExample.setVar errored: Error encoding arguments: Error: hex data is odd-length (argument="value", value="0x100", code=INVALID_ARGUMENT, version=bytes/5.7.0)
```
0x100 is 256 in hex.

--- 

Now what about 2 bytes?

The max number we can represent is 65535 , which is 0xFFFF in hex.

```
contract BytesContract {

    bytes2 n;
    function setVar(bytes2 a) public returns (bytes2) {
        n = a;
        return n;
    }

}
```

If we give it 0x10000 , which is 65536 it returns an error.

---

### Type casting

<https://medium.com/coinmonks/learn-solidity-lesson-22-type-casting-656d164b9991>

- When converting from a larger integer type to a smaller one, the bits on the right are kept while the bits on the left are lost.

```
bytes4 value = 0x12345678;
bytes1 smallValue = bytes1(value); // 0x12
bytes5 largeValue = value; // 0x1234567800;
```

---
