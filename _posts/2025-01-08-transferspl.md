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

So let's start with creating a simple account named CustomPDA

```rust
#[account]
pub struct CustomPda {
    x: u64,
    y: u64
}
```
