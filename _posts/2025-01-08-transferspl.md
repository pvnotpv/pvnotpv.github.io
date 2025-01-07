---
title: Solana - Transfering SPL-tokens from PDA accounts
date: 2025-01-08
categories: [solanaa]
tags: [rust, anchorlang, solana]     
description: Creating associated token account for a PDA and transferring spl tokens out of it.
pin: true
---

---

I've been kinda trying to figure this out for a few days now since there aren't any direct guides out there on how do to do this online. If you just want the final code it's in the end of this post.

---

Here's a quick brief on what we're going to do: 
> First we'll create a custom pda for the program; initialize it with seeds and bump; create an associated token account for it from the program itself and send some spl tokens to it so it can transfer it back; Create a transfer function, when invoked transfers custom amount of spl tokens from the associated token account which is owned by the PDA to the respective target ata. 


So let's start with creating a simple account named CustomPDA, using x and y parameters just for the sake of example.

```rust
#[account]
pub struct CustomPda {
    x: u64,
    y: u64
}
```


