---
name: quasar
description: Build Solana programs using the Quasar framework (quasar-lang / quasar-spl). Use this skill whenever the user mentions Quasar, quasar-lang, quasar_lang, or is building a Solana program with the Quasar framework. Also trigger when the user asks about #[program], #[instruction(discriminator)], Ctx<T>, QuasarSvm, quasar-spl, or the quasar CLI (quasar build, quasar new, quasar test, quasar deploy). If the user is writing any Rust file that imports quasar_lang::prelude::* or quasar_spl, use this skill immediately. Do NOT generate Anchor code — Quasar has a completely different API.
---
 
# Quasar — Zero-Copy Solana Program Framework
 
Quasar is a `no_std` Solana program framework. Accounts are pointer-cast directly from the SVM input buffer — no deserialization, no heap allocation. The syntax resembles Anchor but the generated code and types are completely different. **Never use Anchor types or patterns.**
 
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
 
## Project Structure
 
```
my-program/
├── Cargo.toml
├── Quasar.toml
└── src/
    ├── lib.rs
    ├── state.rs
    ├── event.rs
    ├── errors.rs
    └── instructions/
        ├── mod.rs
        ├── deposit.rs
        └── withdraw.rs
```
 
---
 
## lib.rs — entry point
 
```rust
#![cfg_attr(not(test), no_std)]   // no_std on-chain, std available in tests
 
use quasar_lang::prelude::*;
mod instructions;
use instructions::*;
mod state;
mod event;
 
declare_id!("YourBase58ProgramIdHere111111111111111111111");
 
#[program]
mod my_program {
    use super::*;
 
    #[instruction(discriminator = 0)]
    pub fn deposit(ctx: Ctx<Deposit>, amount: u64) -> Result<(), ProgramError> {
        ctx.accounts.deposit(amount)
    }
 
    #[instruction(discriminator = 1)]
    pub fn withdraw(ctx: Ctx<Withdraw>, amount: u64) -> Result<(), ProgramError> {
        ctx.accounts.withdraw(amount)
    }
}
 
#[cfg(test)]
mod tests;
```
 
Rules:
- `#![cfg_attr(not(test), no_std)]` — not just `#![no_std]`; this lets tests use std.
- `#[program]` goes on a `mod`, not a struct.
- Every handler needs `#[instruction(discriminator = N)]`. No two instructions share the same value.
- First parameter is always `ctx: Ctx<T>` where `T` is the accounts struct name.
- Return type is always `Result<(), ProgramError>`.
 
---
 
## Account Types — QUASAR ONLY
 
Quasar accounts are **references with a `'info` lifetime**, not owned wrappers. Every field in a `#[derive(Accounts)]` struct is a reference.
 
| Use case | Type |
|---|---|
| Signer (mutable) | `&'info mut Signer` |
| Signer (read-only) | `&'info Signer` |
| Unvalidated account | `&'info mut UncheckedAccount` |
| Typed on-chain account (writable) | `&'info mut Account<MyStruct>` |
| Typed on-chain account (read-only) | `&'info Account<MyStruct>` |
| System program | `&'info Program<s>` |
| SPL token mint | `&'info Account<Mint>` |
| SPL token account | `&'info mut Account<Token>` |
| SPL token program | `&'info Program<Token>` |
 
**Never write `Account<'info, T>`, `Signer<'info>`, or `Program<'info, T>` — those are Anchor types and will not compile in Quasar.**
 
---
 
## Accounts Struct (`#[derive(Accounts)]`)
 
```rust
use quasar_lang::prelude::*;
 
#[derive(Accounts)]
pub struct Deposit<'info> {
    pub signer: &'info mut Signer,
 
    #[account(mut, seeds = [b"vault", signer], bump)]
    pub vault: &'info mut UncheckedAccount,
 
    pub system_program: &'info Program<s>,
}
```
 
### Field constraints (on `#[account(...)]`)
 
| Constraint | Meaning |
|---|---|
| `mut` | Account must be writable |
| `seeds = [s, ...]` | Derive PDA and verify address matches |
| `bump` | Auto-derive and verify the canonical bump |
| `bump = field.bump` | Use stored bump instead of re-deriving (cheaper) |
| `init` | Create account via CPI; fails if already exists |
| `init_if_needed` | Create account only if it doesn't exist |
| `payer = <field>` | Which signer pays rent for account creation |
| `space = <expr>` | Byte size when creating |
| `address = <expr>` | Account address must equal expression |
| `has_one = <field>` | `self.<account>.<field> == self.<field>.address()` |
| `constraint = <expr>` | Arbitrary boolean — fails if false |
| `close = <field>` | Close account, send rent lamports to `<field>` |
| `token::mint = <field>` | SPL token account init — set mint |
| `token::authority = <field>` | SPL token account init — set authority |
 
### Address access
 
```rust
// Get the address of any account — the method is .address(), NOT .key()
self.maker.address()      // returns &Address
self.escrow.address()
*self.maker.address()     // dereference to get Address value
```
 
---
 
## On-Chain Account Types (`#[account]`)
 
```rust
use quasar_lang::prelude::*;
 
#[account(discriminator = 1)]
pub struct Escrow {
    pub maker: Address,
    pub mint_a: Address,
    pub mint_b: Address,
    pub maker_token_account_for_mint_b: Address,
    pub receive: u64,
    pub bump: u8,
}
```
 
- `discriminator` must be non-zero.
- Field types: `Address` (32-byte pubkey), `u64`, `u8`, `bool`, or Pod types.
- Use `T::SIZE` in `space =` constraints.
- `set_inner(...)` — positional setter generated for the struct, sets all fields at once:
 
```rust
self.escrow.set_inner(
    *self.maker.address(),
    *self.mint_a.address(),
    *self.mint_b.address(),
    *self.maker_token_account_for_mint_b.address(),
    receive,
    bumps.escrow,
);
```
 
---
 
## Events (`#[event(discriminator = N)]`)
 
```rust
use quasar_lang::prelude::*;
 
#[event(discriminator = 0)]
pub struct MakeEvent {
    pub escrow: Address,
    pub maker: Address,
    pub deposit: u64,
    pub receive: u64,
}
```
 
Emit inside an instruction impl:
 
```rust
emit!(MakeEvent {
    escrow: *self.escrow.address(),
    maker: *self.maker.address(),
    deposit,
    receive,
});
```
 
**`#[event]` without a discriminator does not exist in Quasar. Always provide `discriminator = N`.**
 
---
 
## Errors (`#[error_code]`)
 
```rust
use quasar_lang::prelude::*;
 
#[error_code]
pub enum MyError {
    #[msg("unauthorized")]
    Unauthorized,     // → Custom(3000)
    #[msg("zero amount")]
    ZeroAmount,       // → Custom(3001)
}
```
 
Error codes start at 3000 and increment per variant.
 
Assertion macros:
 
```rust
require!(amount > 0, MyError::ZeroAmount);
require_eq!(a, b, MyError::Unauthorized);
require_keys_eq!(self.escrow.maker, *self.maker.address(), MyError::Unauthorized);
```
 
---
 
## System Program CPI
 
```rust
// Transfer SOL — signer signs directly
self.system_program.transfer(self.signer, self.vault, amount).invoke()?;
 
// Transfer SOL from a PDA — PDA signs via bump seeds
let seeds = bumps.vault_seeds();
self.system_program.transfer(self.vault, self.signer, amount).invoke_signed(&seeds)?;
 
// Direct lamport manipulation for program-owned accounts (no CPI needed)
let vault = self.vault.to_account_view();
let signer = self.signer.to_account_view();
set_lamports(vault, vault.lamports() - amount);
set_lamports(signer, signer.lamports() + amount);
```
 
---
 
## SPL Token CPI (`quasar-spl`)
 
```rust
use quasar_spl::{Mint, Token, TokenCpi};
```
 
```rust
// Transfer tokens — authority signs directly
self.token_program
    .transfer(from_ata, to_ata, authority, amount)
    .invoke()?;
 
// Transfer tokens from PDA-owned account — PDA signs
let seeds = bumps.escrow_seeds();
self.token_program
    .transfer(
        self.vault_token_account_for_mint_a,
        self.dest_ata,
        self.escrow,                                      // PDA as authority
        self.vault_token_account_for_mint_a.amount(),     // .amount() reads token balance
    )
    .invoke_signed(&seeds)?;
 
// Close a token account, send rent to destination
self.token_program
    .close_account(self.vault_token_account_for_mint_a, self.maker, self.escrow)
    .invoke_signed(&seeds)?;
```
 
---
 
## PDA Bump Seeds for Signed CPI
 
When you use `seeds = [b"escrow", maker]` and `bump` in the accounts struct, the macro generates a `<StructName>Bumps` companion (e.g. `MakeBumps`). Bumps are available in instruction impls via the `bumps` parameter:
 
```rust
pub fn make_escrow(&mut self, receive: u64, bumps: &MakeBumps) -> Result<(), ProgramError> {
    // bumps.escrow         → u8, the canonical bump for the escrow PDA
    // bumps.escrow_seeds() → full seed slice for invoke_signed
    let seeds = bumps.escrow_seeds();
    // ...
}
```
 
Always store the bump in state during `init`:
 
```rust
self.escrow.set_inner(..., bumps.escrow);
```
 
On subsequent calls, reuse it with `bump = escrow.bump` instead of re-deriving (saves CUs).
 
---
 
## Tests with `QuasarSvm`
 
Tests use `quasar-svm` — no local validator needed. The client crate is auto-generated by `quasar build`.
 
```rust
// src/tests.rs
extern crate std;
 
use quasar_svm::{Account, Instruction, Pubkey, QuasarSvm};
use solana_address::Address;
use my_program_client::{DepositInstruction, WithdrawInstruction};
 
fn setup() -> QuasarSvm {
    let elf = include_bytes!("../target/deploy/my_program.so");
    QuasarSvm::new().with_program(&Pubkey::from(crate::ID), elf)
}
```
 
### QuasarSvm API
 
```rust
// Pre-load accounts (use for complex test setup)
svm.set_account(Account {
    address: pubkey,
    lamports: 1_000_000_000,
    data: vec![],
    owner: quasar_svm::system_program::ID,
    executable: false,
});
 
// Built-in account helpers
quasar_svm::token::create_keyed_system_account(&pubkey, lamports)
quasar_svm::token::create_keyed_mint_account(&mint_pubkey, &Mint { ... })
quasar_svm::token::create_keyed_associated_token_account(&owner, &mint, token_amount)
 
// Execute instruction
// Pass accounts NOT already loaded via set_account
let result = svm.process_instruction(&instruction, &[account1, account2]);
 
// Chain instructions — result.accounts contains updated state
let result2 = svm.process_instruction(&next_ix, &result.accounts);
 
// Assertions
result.assert_success();
result.assert_error(quasar_svm::ProgramError::Custom(3002));
result.assert_error(quasar_svm::ProgramError::Runtime(String::from("IllegalOwner")));
 
// Read an account from the result
result.account(&pubkey)   // Option<Account>
```
 
### Building client instructions
 
Client instruction structs are named `<InstructionName>Instruction` in PascalCase. Convert to `Instruction` via `.into()`:
 
```rust
let ix: Instruction = DepositInstruction {
    signer: Address::from(user.to_bytes()),
    vault,
    system_program: Address::from(quasar_svm::system_program::ID.to_bytes()),
    amount: 1_000_000_000,
}.into();
 
let result = svm.process_instruction(&ix, &[
    quasar_svm::token::create_keyed_system_account(&user, 10_000_000_000),
    Account {
        address: vault,
        lamports: 0,
        data: vec![],
        owner: crate::ID,
        executable: false,
    },
]);
```
 
---
 
## CLI Commands
 
```bash
quasar init <name>            # scaffold new project
quasar build                  # compile to .so + generate client crate
quasar build --watch          # recompile on change
quasar build --debug          # debug build
quasar test                   # cargo test (builds first)
quasar deploy                 # deploy to cluster in Quasar.toml
quasar new instruction <name> # scaffold a new instruction file
quasar idl                    # generate IDL only
quasar clean                  # clean artifacts
quasar cfg                    # manage global config
```
 
**Always run `quasar build` before `quasar test`** — tests use `include_bytes!` on the compiled `.so`.
 
---
 
## Complete Example — Vault (SOL deposit/withdraw)
 
### src/instructions/deposit.rs
```rust
use quasar_lang::prelude::*;
 
#[derive(Accounts)]
pub struct Deposit<'info> {
    pub signer: &'info mut Signer,
    #[account(mut, seeds = [b"vault", signer], bump)]
    pub vault: &'info mut UncheckedAccount,
    pub system_program: &'info Program<s>,
}
 
impl<'info> Deposit<'info> {
    #[inline(always)]
    pub fn deposit(&mut self, amount: u64) -> Result<(), ProgramError> {
        self.system_program.transfer(self.signer, self.vault, amount).invoke()?;
        Ok(())
    }
}
```
 
### src/instructions/withdraw.rs
```rust
use quasar_lang::prelude::*;
 
#[derive(Accounts)]
pub struct Withdraw<'info> {
    pub signer: &'info mut Signer,
    #[account(mut, seeds = [b"vault", signer], bump)]
    pub vault: &'info mut UncheckedAccount,
}
 
impl<'info> Withdraw<'info> {
    #[inline(always)]
    pub fn withdraw(&self, amount: u64) -> Result<(), ProgramError> {
        let vault = self.vault.to_account_view();
        let signer = self.signer.to_account_view();
        set_lamports(vault, vault.lamports() - amount);
        set_lamports(signer, signer.lamports() + amount);
        Ok(())
    }
}
```
 
---
 
## Complete Example — Escrow (SPL token swap)
 
### src/state.rs
```rust
use quasar_lang::prelude::*;
 
#[account(discriminator = 1)]
pub struct Escrow {
    pub maker: Address,
    pub mint_a: Address,
    pub mint_b: Address,
    pub maker_token_account_for_mint_b: Address,
    pub receive: u64,
    pub bump: u8,
}
```
 
### src/instructions/make.rs
```rust
use quasar_lang::prelude::*;
use quasar_spl::{Mint, Token, TokenCpi};
use crate::state::Escrow;
 
#[derive(Accounts)]
pub struct Make<'info> {
    pub maker: &'info mut Signer,
 
    #[account(init, payer = maker, seeds = [b"escrow", maker], bump)]
    pub escrow: &'info mut Account<Escrow>,
 
    pub mint_a: &'info Account<Mint>,
    pub mint_b: &'info Account<Mint>,
 
    pub maker_token_account_for_mint_a: &'info mut Account<Token>,
 
    #[account(init_if_needed, payer = maker, token::mint = mint_b, token::authority = maker)]
    pub maker_token_account_for_mint_b: &'info mut Account<Token>,
 
    #[account(init_if_needed, payer = maker, token::mint = mint_a, token::authority = escrow)]
    pub vault_token_account_for_mint_a: &'info mut Account<Token>,
 
    pub token_program: &'info Program<Token>,
    pub system_program: &'info Program<s>,
}
 
impl<'info> Make<'info> {
    pub fn make_escrow(&mut self, receive: u64, bumps: &MakeBumps) -> Result<(), ProgramError> {
        self.escrow.set_inner(
            *self.maker.address(),
            *self.mint_a.address(),
            *self.mint_b.address(),
            *self.maker_token_account_for_mint_b.address(),
            receive,
            bumps.escrow,
        );
        Ok(())
    }
 
    pub fn deposit_tokens(&mut self, amount: u64) -> Result<(), ProgramError> {
        self.token_program
            .transfer(
                self.maker_token_account_for_mint_a,
                self.vault_token_account_for_mint_a,
                self.maker,
                amount,
            )
            .invoke()
    }
}
```
 
### src/instructions/refund.rs
```rust
use quasar_lang::prelude::*;
use quasar_spl::{Mint, Token, TokenCpi};
use crate::state::Escrow;
 
#[derive(Accounts)]
pub struct Refund<'info> {
    pub maker: &'info mut Signer,
 
    #[account(
        has_one = maker,
        close = maker,
        seeds = [b"escrow", maker],
        bump = escrow.bump
    )]
    pub escrow: &'info mut Account<Escrow>,
 
    pub mint_a: &'info Account<Mint>,
 
    #[account(init_if_needed, payer = maker, token::mint = mint_a, token::authority = maker)]
    pub maker_token_account_for_mint_a: &'info mut Account<Token>,
 
    pub vault_token_account_for_mint_a: &'info mut Account<Token>,
 
    pub token_program: &'info Program<Token>,
    pub system_program: &'info Program<s>,
}
 
impl<'info> Refund<'info> {
    pub fn withdraw_tokens_and_close(&mut self, bumps: &RefundBumps) -> Result<(), ProgramError> {
        let seeds = bumps.escrow_seeds();
 
        self.token_program
            .transfer(
                self.vault_token_account_for_mint_a,
                self.maker_token_account_for_mint_a,
                self.escrow,
                self.vault_token_account_for_mint_a.amount(),
            )
            .invoke_signed(&seeds)?;
 
        self.token_program
            .close_account(self.vault_token_account_for_mint_a, self.maker, self.escrow)
            .invoke_signed(&seeds)
    }
}
```
 