# Solidity Storage Slot and layout and stuffs

#### A little about hexadecimal

- Hexadecimal uses sixteen distinct symbols, most often the symbols "0"–"9" to represent values 0 to 9, and "A"–"F" (or alternatively "a"–"f") to represent values from ten to fifteen.
- Each hexadecimal digit represents four bits (binary digits), also known as a nibble (or nybble). For example, an 8-bit byte can have values ranging from 00000000 to 11111111 (0 to 255 decimal) in binary form, which can be conveniently represented as 00 to FF in hexadecimal.

8 bits = 11111111 = 255 in decimal = 0xFF in hex

0xF = 1111

0xF = 1111

A single hexadecimal character represents 4 bits.

1 = 0001

2 = 0010

3 = 0011 

4 = 0100

5 = 0101

6 = 0110

7 = 0111

8 = 1000

9 = 1001

A = 1010

B = 1011

C = 1100

D = 1101

E = 1110

F = 1111

1 bytes = 8*1 bits = 8 bits = 1111111 bits = 0xFF 

Which is exactly 2 character long.

32 bytes = 8*32 bits = 256 bits = 11111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111 bits

11111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111 bits when represented in hex = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF

Which is exactly 64 characters in length.

So now we can represent large numbers with less number of characters.

Also a 256-bit unsigned integer can represent up to 115,792,089,237,316,195,423,570,985,008,687,907,853,269,984,665,640,564,039,457,584,007,913,129,639,936 values.

That's like 103 something in length in decimal , better than binary but hexadecimal is better.

---

So I guess now we can get into solidity

Just imagine solidity storage layout as an extremely long boxes chained together and we can store data in those boxes , here is a visual 


|  0  |  1  |  2  |  3  |  4  |  5  |  6  |  7  |  8  |  9  |  10 |  11 | 12  |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |


But how long are these boxes going to be ?, The maximum length of this storage “array” is 2²⁵⁶-1, which means our chained boxes is going to be extremely long , but really how big?
115,792,089,237,316,195,423,570,985,008,687,907,853,269,984,665,640,564,039,457,584,007,913,129,639,936 boxes joined together.

So the next question is how much data can we fit into these boxes?, a maximum of 256 bits or 32 bytes data can be stored in each single box.

Now these boxes are called as slots in solidity. Yeah pretty much the literal meaning.

So let's start with a simple contract. 

```
contract StorageExp {
    uint256 num = 78; 
}
```
Where do you think this num variable is going to be stored?, yes its going to be stored in slot 0; Which is the first slot.

|    78  |
|   ---  |
| Slot0  |

Next lets define another variable, 

```
contract StorageExp {
    uint256 num = 78;
    uint256 rand = 45; 
}
```
The variable rand is going to be stored in the next slot that is slot 1.

|   78   |  45   |
|   ---  |  ---  |
| Slot0  | Slot1 |

Now let's go for a practical explanation. 

Here's a simple contract from Jesper Kristensen video 

```
// SPDX-License-Identifier: MIT
// @author Jesper Kristensen (@cryptojesperk)
pragma solidity 0.8;

contract StorageLayout {
    uint256 x;
    uint128 y;
    uint128 z;

    function set(uint newX, uint128 newY, uint128 newZ) public {
        x = newX;
        y = newY;
        z = newZ;
    }

    // HELPER TO READ FROM STORAGE SLOTS
    function readStorageSlot(uint256 i) public view returns (bytes32 content) {
        assembly {
            content := sload(i) // just a low level function to get the value stored in slots. if i=1 then it outputs the value stored in slot 1.
        }
    }
}
```

We're going to give inputs as 55,23,67 so 
x = 55, y = 23, z = 67

Note here that x is uint256 but y and z are 128 bits each, let's worry about that later; now let's draw the storage layout for x

|    55  | 
|   ---  | 
| Slot0  | 

Now if I run the readStorage() function with input 0 I'm going to get the value stored in slot0, what do you think that's gonna be ?, yeah exactly the value of x.

And it returns 0x0000000000000000000000000000000000000000000000000000000000000037 , but it's showing 37! and what's with all the 0s.

Ok first of all this is a hexadecimal number and now if we represent 55 in hex it's 0x37 , but what about the rest of 0s?

- A maximum of 256 bits can be stored in a slot but all we're doing is just representing the number 55 and it's only a few bits long so what to do with the rest of the space?, yep fill it up with 0s, it's called as padding. Now you may ask why not give the space to other variables but we've already declared the variable x as 256 bits long so it should be 256 bits long, and to make it 256 bits long we just fill it up with 0s.

- Now after coming to the next variables you'll have a lot more deeper understading. As you can see that the variables y and z are 128 bits each and a slot can fit upto 256 bits so why not just divide a single slot and provide half the space to y and the other half to z, yep that's exactly what's going to happen. And just like above we have 128 bits for each variable and it should be 128 bits long but what if our number is just a few bits long?, we fill it up with 0s.

So above we gave 23 for y and 67 for z and if we run readStorageSlot() it's giving us this output
0x0000000000000000000000000000004300000000000000000000000000000017

0x43 = 67 and 0x17 = 23

Hmm looks interesting, isn't this exactly what we wanted? , give half the space to y and for z.
0x00000000000000000000000000000043 which is 32 characters
and
0x00000000000000000000000000000017 which is also 32 characters 

Now the variables being stored in the fast part and last part has something to do with endianess which we're not going to discuss here but I think you can understand what I've said so far.



