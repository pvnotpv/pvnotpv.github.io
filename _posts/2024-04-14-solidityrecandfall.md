---
title: Solidity - Receive and Fallback functions
date: 2024-04-16 
categories: [solidity]
tags: [solidity]     
description: Low level overview of these functions.
---
 
 Which function is called, fallback() or receive()?

           send Ether
               |
         msg.data is empty?
              / \
            yes  no
            /     \
    receive() exists?  fallback()
         /   \
        yes   no
        /      \
    receive()   fallback()



---

- The recieve function is executed when no calldata is provided.
- This function cannot have arguments, cannot return anything and must have external visibility and payable state mutability. It can be virtual, can override and can have modifiers

```
receive() external payable {
      console.log("Me executed when no calldata is provided");
}
```

- Fallback function is executed when none of the other functions match the give function signature.
- In order to receive ether it should be marked as payable.


```
fallback() external payable { 
        console.logBytes(msg.data);
}
```

When the calldata is passed. Through low level interactions like call or through remix ide low level calldata.

```
function callTota(Contract toCallContract) public returns (bool) {
        (bool success,) = address(toCallContract).call(abi.encodeWithSignature("nthada()"));
        return success;
}
```

This nthada() function does not exists so 'console.logBytes(msg.data);' will be executed.

-  If neither a receive Ether nor a payable fallback function is present, the contract cannot receive Ether through a transaction that does not represent a payable function call and throws an exception.
