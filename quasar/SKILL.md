---
name: quasar
description: Build Solana programs using the Quasar framework (quasar-lang / quasar-spl). Use this skill whenever the user mentions Quasar, quasar-lang, quasar_lang, or is building a Solana program with the Quasar framework. Also trigger when the user asks about #[program], #[instruction(discriminator)], Ctx<T>, CtxWithRemaining, QuasarSvm, quasar-spl, or the quasar CLI (quasar build, quasar new, quasar test). If the user is writing any Rust file that imports quasar_lang::prelude::* or quasar_spl, use this skill immediately. Do NOT generate Anchor code — Quasar has a completely different API.
---

# Quasar — Zero-Copy Solana Program Framework

Quasar is a `no_std` Solana program framework. Accounts are pointer-cast directly from the SVM input buffer — no deserialization, no heap allocation, no copies. The syntax resembles Anchor but the types and patterns are completely different. **Never use Anchor types.**

## Reference Files — Read Before Coding

Before writing code for any non-trivial feature, read the relevant reference file:

| Topic | When to read |
|---|---|
| `references/accounts.md` | Account types, constraints, dynamic fields (`String`, `Vec`), sysvars, remaining accounts |
| `references/tokens.md` | quasar-spl, SPL token CPI, Token-2022, ATAs, mint/burn/approve |
| `references/testing.md` | QuasarSvm full API, test patterns, chaining, error matching |
| `references/solana-model.md` | Solana account model, PDAs, rent, CPI mechanics — read this when building anything new |

**If you're unsure which reference applies, read `solana-model.md` first, then the others.**

---

## Cargo.toml

```toml
[package]
name = "my_program"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[features]
alloc = []
client = []

[dependencies]
quasar-lang = { git = "https://github.com/blueshift-gg/quasar" }
quasar-spl  = { git = "https://github.com/blueshift-gg/quasar" }  # only if using SPL tokens

[dev-dependencies]
my_program-client = { path = "target/client/rust/my_program-client" }
quasar-svm        = { git = "https://github.com/blueshift-gg/quasar-svm" }
solana-address    = { version = "2.2.0", features = ["decode"] }
solana-keypair    = "3.1.2"
solana-pubkey     = { version = "4.1.0" }
```

## Quasar.toml

```toml
[project]
name = "my_program"

[toolchain]
type = "solana"

[testing]
framework = "quasarsvm-rust"
```

---

## lib.rs — Program Entry Point

```rust
#![cfg_attr(not(test), no_std)]   // ← always this, not just #![no_std]

use quasar_lang::prelude::*;
mod instructions;
use instructions::*;
mod state;   // #[account] types
mod event;   // #[event] types

declare_id!("YourBase58ProgramIdHere111111111111111111111");

#[program]
mod my_program {
    use super::*;

    // Ctx<T> — standard instruction
    #[instruction(discriminator = 0)]
    pub fn deposit(ctx: Ctx<Deposit>, amount: u64) -> Result<(), ProgramError> {
        ctx.accounts.deposit(amount)
    }

    // CtxWithRemaining<T> — when you need extra accounts at runtime
    #[instruction(discriminator = 1)]
    pub fn create(ctx: CtxWithRemaining<Create>, threshold: u8) -> Result<(), ProgramError> {
        ctx.accounts.create_multisig(threshold, &ctx.bumps, ctx.remaining_accounts())
    }

    // String<N> instruction arg — fixed-max-length string parameter
    #[instruction(discriminator = 2)]
    pub fn set_label(ctx: Ctx<SetLabel>, label: String<32>) -> Result<(), ProgramError> {
        ctx.accounts.update_label(label)
    }
}

#[cfg(test)]
mod tests;
```

Key rules:
- `#![cfg_attr(not(test), no_std)]` — lets tests use std.
- `#[instruction(discriminator = N)]` on every handler. Values must be unique.
- Return type is always `Result<(), ProgramError>`.
- Use `Ctx<T>` normally; `CtxWithRemaining<T>` only when you need extra runtime accounts.

---

## Account Types (QUASAR — not Anchor)

Every accounts struct field is a **reference** with a `'info` lifetime:

```rust
// Standard account reference types:
pub signer:         &'info mut Signer              // writable signer
pub reader:         &'info Signer                  // read-only signer
pub unchecked:      &'info mut UncheckedAccount    // unvalidated, writable
pub my_account:     &'info mut Account<MyStruct>   // typed, writable
pub readonly_acct:  &'info Account<MyStruct>       // typed, read-only

// For accounts with dynamic fields (String/Vec), pass the lifetime through:
pub config:         Account<MultisigConfig<'info>>  // ← note: no & ref, lifetime threaded

// Programs:
pub system_program: &'info Program<s>              // system program
pub token_program:  &'info Program<Token>          // SPL token program

// Sysvars (read reference file for full usage):
pub rent:           &'info Sysvar<Rent>
pub clock:          &'info Sysvar<Clock>
```

**Never write `Account<'info, T>`, `Signer<'info>`, `Program<'info, T>` — those are Anchor and will not compile.**

Address access: `.address()` (never `.key()`)

---

## Accounts Struct — Constraints

```rust
#[derive(Accounts)]
pub struct Make<'info> {
    pub maker: &'info mut Signer,

    #[account(init, payer = maker, seeds = [b"escrow", maker], bump)]
    pub escrow: &'info mut Account<Escrow>,

    #[account(
        has_one = maker,          // escrow.maker == maker.address()
        has_one = maker_ata_b,    // escrow.maker_ata_b == maker_ata_b.address()
        constraint = escrow.receive > 0,
        close = maker,            // close and send rent to maker
        seeds = [b"escrow", maker],
        bump = escrow.bump        // reuse stored bump, saves CUs
    )]
    pub escrow_close: &'info mut Account<Escrow>,

    pub system_program: &'info Program<s>,
}
```

Full constraint table: `mut`, `seeds`, `bump`, `bump = field.bump`, `init`, `init_if_needed`, `payer`, `space`, `address`, `has_one`, `constraint`, `close`, `token::mint`, `token::authority`

---

## On-Chain Account Types

```rust
// Fixed fields only:
#[account(discriminator = 1)]
pub struct Escrow {
    pub maker: Address,
    pub receive: u64,
    pub bump: u8,
}

// With dynamic fields — add a lifetime:
#[account(discriminator = 2)]
pub struct Profile<'a> {
    pub owner: Address,
    pub score: u64,
    pub name: String<'a, 32>,         // String<'lifetime, MAX_BYTES>
    pub tags: Vec<'a, Address, 10>,   // Vec<'lifetime, T, MAX_COUNT>
}
```

Set all fields at once with `set_inner(...)` — positional, generated by the macro:
```rust
self.escrow.set_inner(
    *self.maker.address(),
    receive,
    bumps.escrow,
);
```

For accounts with dynamic fields, `set_inner` also takes a payer and optional rent:
```rust
self.config.set_inner(
    *self.creator.address(),
    threshold,
    bumps.config,
    "",           // label (empty string)
    signers,      // &[Address]
    self.creator.to_account_view(),   // payer for realloc
    Some(&**self.rent),               // rent sysvar
);
```

Auto-generated field accessors: `self.config.label()`, `self.config.signers()`, `self.config.threshold`

---

## Events

```rust
#[event(discriminator = 0)]     // discriminator always required
pub struct MakeEvent {
    pub escrow: Address,
    pub amount: u64,
}
```

```rust
emit!(MakeEvent { escrow: *self.escrow.address(), amount });
```

---

## Errors

```rust
#[error_code]
pub enum MyError {
    #[msg("unauthorized")]
    Unauthorized,     // → ProgramError::Custom(3000)
    #[msg("bad input")]
    BadInput,         // → ProgramError::Custom(3001)
}
```

```rust
require!(amount > 0, MyError::BadInput);
require_eq!(a, b, MyError::Unauthorized);
require_keys_eq!(self.escrow.maker, *self.maker.address(), MyError::Unauthorized);
```

---

## System Program CPI

```rust
// Signer pays:
self.system_program.transfer(self.signer, self.vault, amount).invoke()?;

// PDA pays (uses stored bump seeds):
let seeds = bumps.vault_seeds();
self.system_program.transfer(self.vault, self.recipient, amount).invoke_signed(&seeds)?;

// Direct lamport manipulation (program-owned accounts, no CPI):
let vault  = self.vault.to_account_view();
let signer = self.signer.to_account_view();
set_lamports(vault, vault.lamports() - amount);
set_lamports(signer, signer.lamports() + amount);
```

---

## CLI

```bash
quasar init <n>               # scaffold new project
quasar build                     # compile + generate client crate
quasar build --watch             # watch mode
quasar test                      # cargo test (build first)
quasar deploy                    # deploy per Quasar.toml
quasar new instruction <n>    # scaffold instruction file
quasar idl                       # IDL only
quasar clean                     # clean artifacts
```

Always run `quasar build` before `quasar test` — tests `include_bytes!` the compiled `.so`.
