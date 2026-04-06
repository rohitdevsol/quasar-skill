# SPL Token API — Quasar Reference

Full reference for `quasar-spl`: token accounts, mints, CPI operations, Token-2022, ATAs.

---

## Setup

```toml
# Cargo.toml
[dependencies]
quasar-lang = { git = "https://github.com/blueshift-gg/quasar" }
quasar-spl  = { git = "https://github.com/blueshift-gg/quasar" }
```

```rust
use quasar_lang::prelude::*;
use quasar_spl::{Mint, Token, Token22, TokenCpi, TokenInterface};
```

---

## Account Types

```rust
// SPL Token mint (read-only)
pub mint: &'info Account<Mint>,

// SPL Token-2022 mint (read-only)
pub mint22: &'info Account<Mint22>,

// Token account (writable)
pub token_account: &'info mut Account<Token>,

// Token-2022 account (writable)
pub token22_account: &'info mut Account<Token22>,

// Polymorphic — accepts either Token or Token-2022
pub interface_account: &'info mut Account<TokenInterface>,

// Programs
pub token_program:   &'info Program<Token>,
pub token22_program: &'info Program<Token22>,
```

---

## Mint Accessors

```rust
self.mint.supply()              // u64 — total supply
self.mint.decimals()            // u8
self.mint.mint_authority()      // Option<Address>
self.mint.freeze_authority()    // Option<Address>
self.mint.is_initialized()      // bool
```

---

## Token Account Accessors

```rust
self.token_account.amount()     // u64 — token balance
self.token_account.mint()       // Address
self.token_account.owner()      // Address — the authority
self.token_account.delegate()   // Option<Address>
self.token_account.state()      // AccountState (Initialized, Frozen, etc.)
```

---

## Creating Token Accounts

### ATA (Associated Token Account)

```rust
#[derive(Accounts)]
pub struct InitAta<'info> {
    pub payer: &'info mut Signer,
    pub mint: &'info Account<Mint>,

    #[account(
        init_if_needed,
        payer = payer,
        token::mint = mint,
        token::authority = payer    // wallet owns the ATA
    )]
    pub ata: &'info mut Account<Token>,

    pub token_program: &'info Program<Token>,
    pub system_program: &'info Program<s>,
}
```

### PDA-owned token account (vault)

```rust
#[account(
    init_if_needed,
    payer = maker,
    token::mint = mint_a,
    token::authority = escrow    // PDA owns the vault tokens
)]
pub vault: &'info mut Account<Token>,
```

### Mint init

```rust
#[account(
    init,
    payer = payer,
    mint::decimals = 6,
    mint::authority = payer,
    mint::freeze_authority = payer,   // optional
)]
pub mint: &'info mut Account<Mint>,
```

---

## CPI Operations

Import the `TokenCpi` trait to get CPI builder methods on the program account.

### Transfer

```rust
// Standard transfer — authority signs directly
self.token_program
    .transfer(from_ata, to_ata, authority, amount)
    .invoke()?;

// PDA transfer — PDA signs
let seeds = bumps.escrow_seeds();
self.token_program
    .transfer(
        self.vault,
        self.dest_ata,
        self.escrow,            // PDA as authority
        self.vault.amount(),    // read balance from account
    )
    .invoke_signed(&seeds)?;
```

### Mint to

```rust
self.token_program
    .mint_to(self.mint, self.dest_ata, self.mint_authority, amount)
    .invoke()?;

// Signed
self.token_program
    .mint_to(self.mint, self.dest_ata, self.mint_pda, amount)
    .invoke_signed(&seeds)?;
```

### Burn

```rust
self.token_program
    .burn(self.token_account, self.mint, self.owner, amount)
    .invoke()?;
```

### Approve

```rust
// Grant a delegate permission to spend up to `amount` tokens
self.token_program
    .approve(self.token_account, self.delegate, self.owner, amount)
    .invoke()?;
```

### Revoke

```rust
self.token_program
    .revoke(self.token_account, self.owner)
    .invoke()?;
```

### Close account

```rust
// Close token account, send lamports to destination
self.token_program
    .close_account(self.token_account, self.destination, self.authority)
    .invoke()?;

// PDA authority
let seeds = bumps.escrow_seeds();
self.token_program
    .close_account(self.vault, self.maker, self.escrow)
    .invoke_signed(&seeds)?;
```

### Freeze / Thaw

```rust
self.token_program.freeze_account(self.token_account, self.mint, self.freeze_authority).invoke()?;
self.token_program.thaw_account(self.token_account, self.mint, self.freeze_authority).invoke()?;
```

---

## Token-2022

Everything above works identically for Token-2022 — just use `Token22`, `Mint22`, `Account<Token22>`, and `&'info Program<Token22>` instead.

Polymorphic (accepts both Token and Token-2022):

```rust
pub token_program: &'info Program<TokenInterface>,
pub token_account: &'info mut Account<TokenInterface>,
```

---

## QuasarSvm Test Helpers

```rust
// Create a mint account in tests
quasar_svm::token::create_keyed_mint_account(
    &mint_pubkey,
    &quasar_svm::token::Mint {
        is_initialized: true,
        freeze_authority: COption::None,
        decimals: 6,
        mint_authority: COption::None,
        supply: 100_000_000_000,
    }
)

// Create an ATA with a token balance
quasar_svm::token::create_keyed_associated_token_account(
    &owner,   // &Pubkey — wallet owner
    &mint,    // &Pubkey — mint
    50_000,   // u64 — token balance
)

// Load token program
QuasarSvm::new()
    .with_program(&crate::ID, elf)
    .with_token_program()

// Derive ATA address (test side)
let (ata, _) = Pubkey::find_program_address(
    &[owner.as_ref(), quasar_svm::SPL_TOKEN_PROGRAM_ID.as_ref(), mint.as_ref()],
    &quasar_svm::SPL_ASSOCIATED_TOKEN_PROGRAM_ID,
);

// Read token account state from result
use spl_token::state::Account as SplTokenAccount;
use solana_program_pack::Pack;
let token_state = SplTokenAccount::unpack(
    result.account(&ata).unwrap().data.as_slice()
).unwrap();
assert_eq!(token_state.amount, 5_000_000);
```

Test dev-dependencies for token programs:
```toml
spl-token              = "9.0.0"
solana-program-pack    = "3.1.0"
solana-program-option  = "3.1.0"
```
