---
title: Solana - Transferring SPL-tokens from PDA accounts
date: 2025-01-07
categories: [solana]
tags: [rust, anchorlang, solana]     
description: Creating an associated token account for a PDA and transferring SPL tokens out of it.
pin: true
---

---

So I've been kind of trying to figure this out for a few days now since there aren't any direct guides out there on how to do this online. If you just want the final code, it's at the end of this post.

---

Here's a quick brief on what we're going to do: 
> First we'll create a custom pda for the program; initialize it with seeds and bump; create an associated token account for it from the program itself and send some spl tokens to it so it can transfer it back; Create a transfer function, when invoked transfers custom amount of spl tokens from the associated token account which is owned by the PDA to the respective target ata. 

---

## Creating a new spl token and minting some tokens.

```bash
~> spl-token create-token
Creating token 12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ under program TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA

Address:  12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ
Decimals:  9

Signature: 56vk65kjfigu8ydovrMpN92QzSbLY1FED72Czpd7snGxh8Dz2cmhpjefnpyLzJJ7VyHKhVbmLTEqhazcYecavUhz
```

Now let's create an associated token account for our account and mint some spl tokens.

```bash
~> spl-token create-account 12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ
Creating account GEKodQzJAewUBJSS1H4jQZL7a1xB4d3yLMqzZw5faaKM

Signature: 5u9DMRzhenMGS3jpi5L2rUeWgFhh6J4X86CiVuBBMuHdFXv28Mz2C5HucyUfis3jqZxwVoLGeTMStBqNioh81P37
```
Minting some spl for the ata:

```bash
~> spl-token mint 12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ 100000
Minting 100000 tokens
  Token: 12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ
  Recipient: GEKodQzJAewUBJSS1H4jQZL7a1xB4d3yLMqzZw5faaKM

Signature: 2XqVc7WkYXH83EKBUEiPPas8xfqj7UPMi5jLwbpzzXHXrJ19YExbkeB13XXWAyjQbuHeXchYRdW8ReCRnG7iQqmi
```

---

## Initializing anchor project for the POC

#### Creating a PDA account which will further have a associated token account for the particular mint.

```rust
#[account]
pub struct CustomPda {
    x: u64,
    y: u64
}
```

#### Creating a init-transfer function that outputs the address of the newly created PDA and it's particular ATA.

> Note: The init-transfer function is just for initializing the account and printing the ATA so that we can transfer some tokens to it at first, we'll also create another function that does the actual transfer. Also both the functions will the use the same reference struct named <TransferSol>

```rust
pub fn init_transfer(ctx: Context<TransferSpl>) -> Result<()> {
    msg!("The pda {}", ctx.accounts.transfer_pda.key());
    msg!("The token ata is {}", ctx.accounts.token_ata.key());
    Ok(())
}
```

###### The TransferSol Struct:

- Initializing the PDA
- Creating a token ATA for the PDA
- Specifying the recipient account and recipient's ATA.

```rust
#[derive(Accounts)]
pub struct TransferSpl<'info> {
    #[account(
        init_if_needed,
        payer = signer,
        space = size_of::<CustomPda>() + 8,
        seeds = [b"transfer".as_ref()],
        bump
    )]
    pub transfer_pda: Account<'info, CustomPda>,

    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,

    #[account(
        init_if_needed,
        payer = signer,
        associated_token::mint = mint,
        associated_token::authority = transfer_pda
    )]
    pub token_ata: Account<'info, TokenAccount>,

    pub to_owner: SystemAccount<'info>,
    #[account(
        mut,
        token::mint = mint,
        token::authority = to_owner
    )]
    pub to_ata: Account<'info, TokenAccount>,

    pub mint: Account<'info, Mint>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub token_program: Program<'info, Token>

}
```

- We'll first call this function to get the Associated Token Account for the PDA and transfer tokens to it.

```javascript
const user = new anchor.web3.PublicKey("6i6Hf1PdcDk4Yt5dbwk4yL1qdspffZSDwW4wK5n3tezC");
const mintAccount = new anchor.web3.PublicKey("12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ")
const toAcc = new anchor.web3.PublicKey("GEKodQzJAewUBJSS1H4jQZL7a1xB4d3yLMqzZw5faaKM")

const initVault = await program.methods.initTransfer()
  .accounts({
      mint: mintAccount,
      toOwner: user,
      toAta: toAcc
  }).rpc();
console.log(initVault)
```

TX hash after running: ```4AYvHT2ijGL4q5LpHFRzSoFXdFWhEtKtKrjX9QNaArbFDGkomPcLUfxj8N1nd7qpWash2vvTXTPH4efCDjw1UAB2```, either use <explorer.solana.com> to get the accounts from logs or use the solana cli.

```bash
 solana confirm -v 4AYvHT2ijGL4q5LpHFRzSoFXdFWhEtKtKrjX9QNaArbFDGkomPcLUfxj8N1nd7qpWash2vvTXTPH4efCDjw1UAB2
```

Output:

```
Program ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL success
Program log: The pda Femc9tsWKC1eCiQT6BbzcBqXS5nsPM6jbxyKBBJKuai
Program log: The token ata is FKLyGQDqFAFAy9kuVAftSV6eUS2seCf7YzYAyPZSobuz
```

#### If we use spl account-info for the ATA, we can see that the ATA is owned by the PDA , which is exactly what we wanted:

```bash
~> spl-token account-info --address FKLyGQDqFAFAy9kuVAftSV6eUS2seCf7YzYAyPZSobuz

SPL Token Account
  Address: FKLyGQDqFAFAy9kuVAftSV6eUS2seCf7YzYAyPZSobuz
  Program: TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
  Balance: 0
  Decimals: 9
  Mint: 12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ
  Owner: Femc9tsWKC1eCiQT6BbzcBqXS5nsPM6jbxyKBBJKuai
  State: Initialized
  Delegation: (not set)
  Close authority: (not set)
```

#### Transferring some spl tokens to the ATA of the PDA.

```bash
~> spl-token transfer 12a1jHHLKTNjiHgPu1zeZs9Gvn1dvuHXGUzayGaRKjYJ 1000 FKLyGQDqFAFAy9kuVAftSV6eUS2seCf7YzYAyPZSobuz
Transfer 1000 tokens
Sender: GEKodQzJAewUBJSS1H4jQZL7a1xB4d3yLMqzZw5faaKM
Recipient: FKLyGQDqFAFAy9kuVAftSV6eUS2seCf7YzYAyPZSobuz

Signature: 2xHzt5gU2nVMMX4fn1ANLEbbB42u7cb6fw9EMPmujoLBi3MBcXWvkEfRWdfUkvxm8XiAwTZeN2xMP4gfeyg349R4
```

#### Creating the main TransferSPL function

```rust
pub fn withdraw(ctx: Context<TransferSpl>, amount: u64) -> Result<()> {
        let (pda, _bump) = Pubkey::find_program_address(
            &[b"transfer".as_ref()],
            ctx.program_id
        );

        msg!("The pda of the account after deriving is {}", pda);

        let bump_seed = ctx.bumps.transfer_pda;
        let signer_seeds : &[&[&[u8]]] = &[&[b"transfer".as_ref(), &[bump_seed]]];

        let cpi_context = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(), 
            Transfer {
                from: ctx.accounts.token_ata.to_account_info(),
                to: ctx.accounts.to_ata.to_account_info(),
                authority: ctx.accounts.transfer_pda.to_account_info(),
            }, 
            signer_seeds
        );

        transfer(cpi_context, amount)?;

        Ok(())
}
```

THE MOST IMPORTANT THING HERE TO NOTE IS THAT: THE AUTHORITY FOR CROSS PROGRAM INSTRUCTION SHOULD BE THE PDA, since the PDA is created by using the public key of the present program, the CPI works. For the ones curious I added a snippet to the code on how this works/how the public key derivation of the pda works under the hood, If you want an exaplanation of the snippet, I've provided a stackexchange answer below.

The snippet:
```rust
let (pda, _bump) = Pubkey::find_program_address(
    &[b"transfer".as_ref()],
    ctx.program_id
);

msg!("The pda of the account after deriving is {}", pda);
```

<https://solana.stackexchange.com/a/18843>

> This snippet is just optional , the output of the message is going to be the pda, which we created. So as you have figured it out, the signer seeds just basically derives the PDA , again since the program's public key is used in the signer seeds when creating the PDA, the signing part works.

### Calling the final function

```javascript
const withdraw = await program.methods.withdraw(new anchor.BN(90000000000)) // 9 decimals
      .accounts({
        mint: mintAccount,
        toOwner: user,
        toAta: toAcc
      }).rpc()

console.log(withdraw)
```

#### The Program transfers the particular amount of spl tokens from the PDA to the desired account, which is exactly what we wanted!!!
