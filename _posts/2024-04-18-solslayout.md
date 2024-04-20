---
title: Solidity Storage - Slots and layout
date: 2024-04-18
categories: [solidity]
tags: [solidity,solidityslots,soliditylayout]     
description: In depth explanation of solidity storage and layout starting from the basics of hexadecimal.
pin: true
---

Sources I've used to learn this topic.

- <https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html>
- <https://www.youtube.com/watch?v=i_LwhlFNSkI&t=1817>
- <https://medium.com/@flores.eugenio03/exploring-the-storage-layout-in-solidity-and-how-to-access-state-variables-bf2cbc6f8018>
- <https://coinsbench.com/solidity-layout-and-access-of-storage-variables-simply-explained-1ce964d7c738>
- <https://docs.alchemy.com/docs/smart-contract-storage-layout>

Now the sole reason why I decided to make this post is becauase there was some stuffs I found hard to understand like the stuffs with hexadecimal, padding and etc. 
Also it's highly recommended that you learn stuffs from multiple sources and tutors , never let alone from just a single source/tutor.

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

- Which is exactly 2 character long in hex.

32 bytes = 8*32 bits = 256 bits = 11111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111 bits

11111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111 bits when represented in decimal = 115,792,089,237,316,195,423,570,985,008,687,907,853,269,984,665,640,564,039,457,584,007,913,129,639,936

- That's pretty huge number but see that we've decreased the character length from the 256 to a 103ish , now what if we convert that number to hex? 
Which is  0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF

- Exactly 64 characters in length in hex.

So now we can represent large numbers with less number of characters.

(If you still don't understand, google how to convert binary to decimal and binary to hex; you'll be pertty clear then)

---

So I guess now we can get into solidity

Just imagine solidity storage layout as boxes chained together and we can store data in those boxes , here is a visual 

![slots](/images/table.png)


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

```
pragma solidity 0.8;

contract StorageLayout {
    uint256 x;
    uint256 y;

    function setVar(uint256 a, uint256 b) public {
        x = a;
        y = b;
    }

    // Below function is from Jesper Kristensen's video

    function readStorageSlot(uint256 i) public view returns (bytes32 content) {
        assembly {
            content := sload(i) // just a low level function to get the value stored in slots. if i=1 then it outputs the value stored in slot 1.
        }
    }
}
```

We're going to give inputs as 55,23 so 
x = 55, y = 23 

Storage layout for x and y:

|    55  |  23   |
|   ---  | ---   |
| Slot0  | Slot1 |

Now if I run the readStorage() function with input 0 I'm going to get the value stored in slot0, what do you think that's gonna be ?, yeah exactly; the value of x.

And it returns 0x0000000000000000000000000000000000000000000000000000000000000037 , but it's showing 37! and what's with all the 0s.

Ok first of all this is a hexadecimal number and now if we represent 55 in hex it's 0x37 , but what about the rest of 0s?

- A maximum of 256 bits can be stored in a slot but all we're doing is just representing the number 55 and it's only a few bits long so what to do with the rest of the space?, yep fill it up with 0s, it's called as padding. Now you may ask why not give the space to other variables but we've already declared the variable x as 256 bits long so it should be 256 bits long, and to make it 256 bits long we just fill it up with 0s.

Same goes for the variable y , it returns 0x0000000000000000000000000000000000000000000000000000000000000017 which is 23 in decimal.

Aight , now let's declare another two variables named 'z' and 't' but this time let's declare them as 128 bits long. 

```
pragma solidity 0.8;

contract StorageLayout {
    uint256 x;
    uint256 y;
    uint128 z;
    uint128 t;

    function setVar(uint256 a, uint256 b) public {
        x = a;
        y = b;
    }

    function setnewVar(uint128 a, uint128 b) public {
        z = a;
        t = b;
    }

    function readStorageSlot(uint256 i) public view returns (bytes32 content) {
        assembly {
            content := sload(i) // just a low level function to get the value stored in slots. if i=1 then it outputs the value stored in slot 1.
        }
    }
}
```
The new variables can store a max 128 bit number and which is the max 128 bit number in decimal ? 
111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111 = 680564733841876926926749214863536422911 in decimal, I'm not really how accurate the conversion is but you can see that it's a pretty big number. 

128 + 128 = 256

Now the doubt you've probably had earlier , why wasting space by filling up with zeros? can't we instead like somehow decrease the size of the variable or just store multiple variables in a single slot. Yep that's what we just did, we used a 128bit variable.

- As you can see that the variables z and t are 128 bits each and a slot can fit upto 256 bits so why not just divide a single slot and provide half the space to z and the other half to t, yep that's exactly what's going to happen. But wait what if our number is just a few bits long?. Oops looks like we again gotta fill it up with zeroes but either way it's better than before.

So if we give 23 for z and 67 for t and if we run readStorageSlot() it's giving us this output
0x0000000000000000000000000000004300000000000000000000000000000017

0x43 = 67 and 0x17 = 23

Hmm looks interesting, isn't this exactly what we wanted? , give half the space to 'z' and other half to 't' of a single slot.
0x00000000000000000000000000000043 which is 32 characters
and
0x00000000000000000000000000000017 which is also 32 characters 

Now the variables being stored in the first part and last part has something to do with endianess which we're not going to discuss here but I think you can understand what I've said so far.

---

Now let's dive into mappings 

A quick recap of mappings before it's layout:
Instead of an explanation I'll show you an example 

```
mapping(address => uint) balance;
balance[0x5B38Da6a701c568545dCfcB03FcB875f56beddC4] = 10;
return balance[0x5B38Da6a701c568545dCfcB03FcB875f56beddC4];
```

And it returns the balance of the address which is 10.

---

Now let's get into the storage part with an example contract. The source code is from Jesper Kristensen's <https://www.youtube.com/watch?v=i_LwhlFNSkI&t=1817> video with slight modifications.

```
// SPDX-License-Identifier: MIT
// @author Jesper Kristensen (@cryptojesperk)
pragma solidity 0.8;

contract StorageLayout {
    uint x = 2;
    mapping(uint => uint) acc;
    uint y = 3;

    function addToM(uint key, uint value) public {
        acc[key] = value;
    }

    function readStorageSlot(uint256 i) public view returns (bytes32 content) {
        assembly {
            content := sload(i)
        }
    }

    function getLocationOfMapping(uint mappingSlot, uint key) public pure returns (uint slot) {
        return uint256(keccak256(abi.encode(key, mappingSlot)));
    }
}

```

As we can see that the value in slot0 is going to be 2 because it's the first declared. 
And now what about the next slot? which is not a single variable but a mapping. So the thing about mapping is that it's size is increased dynamically meaning; just think about the variable 'y' which is declared after the mapping it's obviously going to be stored in the next slot after mapping which is slot2. What do you think is gonna happen if we add a key-value pair to the mapping, is the slot going to be pushed; in the sense like the variable 'y' to slot3? No. Solidity has an amazing solution for this problem. 

Ok let's start with an example, acc[3] = 7;

(acc is currently in slot1, the key is 3 and the value is 7)

This 7 is going to be stored in a slot number after a particular operation. 

Which is the key is hased with the particular slot.

```
function getLocationOfMapping(uint mappingSlot, uint key) public pure returns (uint slot) {
        return uint256(keccak256(abi.encode(key, mappingSlot)));
}
```
If we provide mappingSlot with 1 (which is the mapping of acc) and key as 3 , we get this output '56988696150268759067033853745049141362335364605175666696514897554729450063371' , this is the place where the value 7 is stored. Interesting isn't it.

Next let's add another key-value pair acc[4] = 2

We get the output as '107553882524790531947385985832592837884442228935463780553192851707863573624387' , which has no relation with the previous slot even though we just increased the value of key by one.

---

### Nested mappings

A simple bank contract that allows users to store money in different banks with the same account. (Just for logical understanding of nested mappings)

```
contract Banks {
    mapping(uint => mapping(address => uint)) accountNumber;

    function setBalance(uint bankNumber, address _addr, uint balance) public {
        accountNumber[bankNumber][_addr] = balance;
        
    }

    function getBalance(uint bankNumber, address _addr) view public returns (uint) {
        return accountNumber[bankNumber][_addr];
    }

}
```

Now below is an example contract to explain about the location of nested mappings.

```
pragma solidity 0.8;

contract Banks {
    mapping(uint => mapping(address => uint)) accountNumber;

    function setBalance(uint bankNumber, address _addr, uint balance) public {
        accountNumber[bankNumber][_addr] = balance;
        
    }

    function getBalance(uint bankNumber, address _addr) view public returns (uint) {
        return accountNumber[bankNumber][_addr];
    }

    function readStorageSlot(uint256 i) public view returns (bytes32 content) {
        assembly {
            content := sload(i)
        }
    }

    function getLocationOfAddress(uint mappingSlot, uint bankNumber) public pure returns (uint slot) {
        return uint256(keccak256(abi.encode(bankNumber, mappingSlot)));
    }

    function getLocationOfValue(uint mappingSlot, address _addr) public pure returns (uint slot) {
        return uint256(keccak256(abi.encode(_addr, mappingSlot)));
    }

}

```

accountNumber[bankNumber][_addr] = balance;

Now we're going to call setBalance with (100, 0xCc8188e984b4C392091043CAa73D227Ef5e0d0a7, 500)

;The account has 500 balance in the bank with number 100.

So first of all in which slot is the mapping stored?, yes slot0.

Now if we hash the bankNumber with slot0 we get location of where the new mapping is stored.

Calling getLocationOfAddress(0, 100) returns 71336474783394080197321810469482573231222347487862131686619557077352214456178 which is the slot where the new mapping is stored.

So it more like restarts the cycle, now we need to call getLocationOfValue(0xCc8188e984b4C392091043CAa73D227Ef5e0d0a7, 71336474783394080197321810469482573231222347487862131686619557077352214456178) which finally returns the value.

# Todo
<https://medium.com/@dariusdev/how-to-read-ethereum-contract-storage-44252c8af925>
---

Continue: <https://pvnotpv.github.io/posts/endianness/>

