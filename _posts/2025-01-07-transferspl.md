---
title: Solana - Transfering SPL-tokens from PDA accounts
date: 2025-01-07
categories: [solanaa]
tags: [rust, anchorlang, solana]     
description: Creating associated token account for a PDA and transferring spl tokens out of it.
pin: true
---

---

I've been kinda trying to figure this out for a few days now since there aren't any direct guides out there on how do to do this online. If you just want the final code it's in the end of the post.

---

Here's a quick brief on what we're going to do: 
> First we'll create a custom pda for the program; initialize it with seeds and bump; create an associated token account for it from the program itself and send some spl tokens to it so it can transfer it back; Create a transfer function, when invoked transfers custom amount of spl tokens from the associated token account which is owned by the PDA to the respective target ata. 

---

Let's start by creating a new spl token using the command line tool.

```
~> spl-token create-token
Creating token Grg9sGgXYF6ej36AxP75Lm7P2UG3S4peEUTGQBom4Uoa under program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA

Address:  Grg9sGgXYF6ej36AxP75Lm7P2UG3S4peEUTGQBom4Uoa
Decimals:  9

Signature: 49vU6jSZEBia7tBfYHZoGFTzs9VeVxHE2nv7ThXDSocUmHRwfKNSnkk1JmkNeFXKHST6byAGYynXRLkCefxTQ5ki
```

Now let's create an associated token account for our account and mint some spl tokens.

```
~> spl-token create-account Grg9sGgXYF6ej36AxP75Lm7P2UG3S4peEUTGQBom4Uoa
Creating account DoRvQ2vvuUrr73DP7HVj9bCujPHBMpaagYtQffi3iq8V

Signature: 57nbESbfqBRX7vzHTghU2sasRx6PJJGbnvoWdvuk68u2cThZhaEuckabEbQJbuQAZVyfjH8afWm5TAXHS96SGyex
```
Minting some spl for the ata:

```
~> spl-token mint Grg9sGgXYF6ej36AxP75Lm7P2UG3S4peEUTGQBom4Uoa 100000
Minting 100000 tokens
  Token: Grg9sGgXYF6ej36AxP75Lm7P2UG3S4peEUTGQBom4Uoa
  Recipient: DoRvQ2vvuUrr73DP7HVj9bCujPHBMpaagYtQffi3iq8V

Signature: 2fFYZ6G3njH7pFF75LbLgFcJ4Uff3MbijsfiNb3F3ZuYcQnTaSzWV5WF3Fhcjb44nGEq9AGogQk7JfjFaDKCTHnF
```

---

Now let's create a new program using anchor init.
```rust
#[account]
pub struct CustomPda {
    x: u64,
    y: u64
}
```


s s


