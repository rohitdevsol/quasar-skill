---
name: quasar
description: Build Solana programs using the Quasar framework (quasar-lang). Use this skill whenever the user mentions Quasar, quasar-lang, quasar_lang, a zero-copy Solana program, or is building a Solana program that isn't using Anchor. Also trigger when the user asks about #[program], #[instruction(discriminator)], Ctx<T>, QuasarSvm, or the quasar CLI commands (quasar build, quasar new, quasar test, quasar deploy). If the user is writing any Rust Solana program with quasar_lang::prelude::*, use this skill immediately.
---
 
# Quasar — Zero-Copy Solana Program Framework
 
Quasar is a Rust framework for Solana programs. It provides Anchor-compatible ergonomics with lower compute unit overhead by using zero-copy pointer casts to `#[repr(C)]` companion structs — no heap allocation, no deserialization.
 
---
 
## Project Structure
 
```
my-program/
├── Quasar.toml           # project config (toolchain, client languages)
├── src/
│   ├── lib.rs            # #[program] entry point
│   ├── state.rs          # #[account] types
│   ├── errors.rs         # #[error_code] enum
│   └── instructions/
│       ├── mod.rs
│       ├── deposit.rs
│       └── withdraw.rs
└── client/               # auto-generated from IDL
    └── src/lib.rs
```
 
---
 
## Core Imports
 
Every program file starts with:
 
```rust
#![no_std]
use quasar_lang::prelude::*;
```
 
---
 
## Defining the Program
 
```rust
declare_id!("11111111111111111111111111111111");
 
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
```
 
Key rules:
- `#[program]` goes on a `mod`, not a struct.
- Every handler needs `#[instruction(discriminator = N)]` — no two instructions share the same discriminator. Discriminators can be a single integer (`= 0`) or a byte array (`= [1, 0, 0, 0]`).
- First parameter is always `ctx: Ctx<T>` where `T` is the accounts struct.
- Return type is always `Result<(), ProgramError>`.
 
---
 
## Accounts Struct (`#[derive(Accounts)]`)
 
```rust
#[derive(Accounts)]
pub struct Deposit<'info> {
    #[account(mut, signer)]
    pub user: UncheckedAccount<'info>,
 
    #[account(
        mut,
        seeds = [b"vault", user.key().as_ref()],
        payer = user,
        space = Vault::SIZE,
    )]
    pub vault: Account<'info, Vault>,
 
    pub system_program: Program<'info, System>,
}
```
 
### Field attributes
 
| Attribute | Meaning |
|-----------|---------|
| `mut` | Account must be writable |
| `signer` | Account must have signed the transaction |
| `address = <expr>` | Account address must equal `<expr>` |
| `seeds = [s, ...]` | Derive PDA and verify address matches |
| `init` | Create account via CPI (fails if already exists) |
| `init_if_needed` | Create account only if it doesn't exist |
| `payer = <field>` | Which signer pays for account rent |
| `space = <expr>` | Byte size for account creation |
 
### Account wrapper types
 
| Type | Checks |
|------|--------|
| `Account<'info, T>` | Owner, discriminator, size |
| `Signer<'info>` | Must be a signer |
| `UncheckedAccount<'info>` | No automatic checks — validate manually |
| `Program<'info, T>` | Must be executable, matches known ID |
| `SystemAccount<'info>` | Owned by system program |
 
---
 
## On-Chain Account Types (`#[account]`)
 
```rust
#[account(discriminator = 1)]
pub struct Vault {
    pub owner: Address,
    pub amount: u64,
    pub bump: u8,
}
```
 
- `discriminator` must be non-zero.
- Generates a zero-copy companion struct `ZcVault` with `#[repr(C)]` layout and alignment-1 `Pod*` fields.
- Implements `Discriminator`, `Space`, `Owner`, `AccountCheck`, `ZeroCopyDeref`.
- Use `T::SIZE` for `space =` in account constraints.
 
### Pod integer types (use inside `#[account]` structs for zero-copy correctness)
 
`PodU16`, `PodU32`, `PodU64`, `PodU128`, `PodI16`, `PodI32`, `PodI64`, `PodI128`, `PodBool`
 
---
 
## Errors (`#[error_code]`)
 
```rust
#[error_code]
pub enum MyError {
    #[msg("amount must be greater than zero")]
    InvalidAmount,
    #[msg("vault is locked")]
    VaultLocked,
}
```
 
---
 
## Events (`#[event]`)
 
```rust
#[event]
pub struct DepositEvent {
    pub user: Address,
    pub amount: u64,
}
```
 
Emit inside a handler:
 
```rust
emit!(DepositEvent { user: *ctx.accounts.user.key(), amount });
```
 
---
 
## Constraint Macros
 
```rust
require!(amount > 0, MyError::InvalidAmount);
require_eq!(vault.owner, *user.key(), MyError::Unauthorized);
require_keys_eq!(vault.mint, mint.key(), MyError::WrongMint);
```
 
---
 
## PDAs
 
```rust
// In accounts struct — Quasar derives and verifies automatically:
#[account(seeds = [b"vault", user.key().as_ref()])]
pub vault: Account<'info, Vault>,
 
// Access the bump from ctx:
let bump = ctx.bumps.vault;
 
// Manual verification (lower-level):
verify_program_address(&[b"vault", user.key().as_ref(), &[bump]], &crate::ID, &vault.key())?;
```
 
---
 
## CPI (Cross-Program Invocation)
 
```rust
// System program transfer
System::transfer(
    ctx.accounts.from.to_account_view(),
    ctx.accounts.to.to_account_view(),
    amount,
)?;
 
// With PDA signer seeds
System::transfer_signed(
    ctx.accounts.vault.to_account_view(),
    ctx.accounts.user.to_account_view(),
    amount,
    &[Seed::from(b"vault"), Seed::from(user.key().as_ref()), Seed::from(&[bump])],
)?;
```
 
---
 
## Tests with `QuasarSvm`
 
Tests run in-process against `QuasarSvm` — no local validator needed.
 
```rust
extern crate std;
use {
    quasar_svm::{Account, Instruction, Pubkey, QuasarSvm},
    my_program_client::*,
    std::println,
};
 
fn setup() -> QuasarSvm {
    let elf = std::fs::read("../../target/deploy/my_program.so").unwrap();
    QuasarSvm::new().with_program(&crate::ID, &elf)
}
 
#[test]
fn test_deposit() {
    let mut svm = setup();
    let user = Pubkey::new_unique();
    let (vault, _) = Pubkey::find_program_address(&[b"vault", user.as_ref()], &crate::ID);
 
    let ix: Instruction = DepositInstruction {
        user,
        vault,
        system_program: quasar_svm::system_program::ID,
        amount: 1_000_000_000,
    }.into();
 
    let result = svm.process_instruction(&ix, &[
        quasar_svm::token::create_keyed_system_account(&user, 10_000_000_000),
        Account { address: vault, lamports: 0, data: vec![], owner: quasar_svm::system_program::ID, executable: false },
    ]);
 
    assert!(result.is_ok(), "{:?}", result.raw_result);
    println!("CU: {}", result.compute_units_consumed);
}
```
 
For token programs:
 
```rust
QuasarSvm::new()
    .with_program(&crate::ID, &elf)
    .with_token_program()
```
 
---
 
## CLI Commands
 
```bash
quasar init <name>          # scaffold a new project
quasar build                # compile to BPF (.so)
quasar build --watch        # recompile on file change
quasar build --debug        # debug build
quasar test                 # run tests (cargo test under the hood)
quasar deploy               # deploy to cluster in Quasar.toml
quasar new instruction <name>  # scaffold a new instruction file
quasar idl                  # generate IDL
quasar clean                # clean build artifacts
quasar cfg                  # manage global config
```
 
---
 
## Quasar.toml
 
```toml
[toolchain]
type = "solana"      # or "native" for sbpf-linker
 
[client]
languages = ["rust", "typescript"]
```
 
---
 
## Complete Minimal Example
 
```rust
// src/lib.rs
#![no_std]
use quasar_lang::prelude::*;
 
mod instructions;
use instructions::*;
mod state;
 
declare_id!("YourProgramIdHere1111111111111111111111111");
 
#[program]
mod my_program {
    use super::*;
 
    #[instruction(discriminator = 0)]
    pub fn initialize(ctx: Ctx<Initialize>, amount: u64) -> Result<(), ProgramError> {
        ctx.accounts.initialize(amount, &ctx.bumps)
    }
}
```
 
```rust
// src/state.rs
use quasar_lang::prelude::*;
 
#[account(discriminator = 1)]
pub struct MyAccount {
    pub owner: Address,
    pub amount: PodU64,
    pub bump: u8,
}
```
 
```rust
// src/instructions/initialize.rs
use quasar_lang::prelude::*;
use crate::state::MyAccount;
 
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut, signer)]
    pub user: UncheckedAccount<'info>,
 
    #[account(
        init,
        seeds = [b"account", user.key().as_ref()],
        payer = user,
        space = MyAccount::SIZE,
    )]
    pub my_account: Account<'info, MyAccount>,
 
    pub system_program: Program<'info, System>,
}
 
impl<'info> Initialize<'info> {
    pub fn initialize(&mut self, amount: u64, bumps: &InitializeBumps) -> Result<(), ProgramError> {
        let acc = self.my_account.as_mut();
        acc.owner = *self.user.key();
        acc.amount = amount.into();
        acc.bump = bumps.my_account;
        Ok(())
    }
}
```
 
---
 
## Key Differences from Anchor
 
| | Anchor | Quasar |
|---|---|---|
| Account access | Borsh deserialization | Zero-copy pointer cast |
| Context type | `Context<T>` | `Ctx<T>` |
| Program macro | `#[program]` on mod | `#[program]` on mod |
| Discriminators | Auto-generated from name hash | Explicit `discriminator = N` |
| Testing | LiteSVM / bankrun | `QuasarSvm` (built-in) |
| `no_std` | No | Yes |
| Address type | `Pubkey` | `Address` |
| Int types | `u64` directly | `PodU64` in account structs |
 