# Accounts API — Quasar Reference

Full reference for `#[derive(Accounts)]`, account types, constraints, dynamic fields, sysvars, and remaining accounts.

---

## All Account Field Types

Account fields do not use lifetimes. Never use Anchor-style owned wrappers.
Mark writability with `#[account(mut)]` on the field.

```rust
// Signers
#[account(mut)]
pub signer:     Signer                      // must sign + writable
pub reader:     Signer                      // must sign, read-only

// Unvalidated
#[account(mut)]
pub unchecked:  UncheckedAccount            // writable, no checks — validate manually
pub readonly:   UncheckedAccount            // read-only, no checks

// Typed (fixed fields only)
#[account(mut)]
pub account:    Account<MyStruct>
pub readonly:   Account<MyStruct>

// Typed with dynamic fields (String/Vec)
pub config:     Account<MultisigConfig>      // writable (default when init/init_if_needed)
pub config_ro:  Account<MultisigConfig>      // read-only (no mut constraint)

// Programs
pub system_program: Program<System>          // Solana system program
pub token_program:  Program<Token>           // SPL token
pub token22_program: Program<Token22>        // Token-2022

// Sysvars
pub rent:   Sysvar<Rent>
pub clock:  Sysvar<Clock>

// SPL token accounts (from quasar-spl)
pub mint:   Account<Mint>                     // read-only mint
#[account(mut)]
pub token:  Account<Token>                    // writable token account
```

Address access for any type: `.address()` — returns `&Address`.
Never use `.key()` (Anchor).

---

## Full Constraint Reference

Constraints go in `#[account(...)]` on individual fields.

### Basic

| Constraint | Effect |
|---|---|
| `mut` | account must be writable |
| `address = <expr>` | account's address must equal `<expr>` |

### PDA

| Constraint | Effect |
|---|---|
| `seeds = [s, ...]` | derive PDA from seeds, verify address matches |
| `bump` | auto-derive canonical bump, verify |
| `bump = field.bump` | use stored bump (cheaper — avoids re-derivation) |

Seeds can be: `b"literal"`, field names (expands to `.address()`), byte slices.

### Creation

| Constraint | Effect |
|---|---|
| `init` | create account via CPI; fails if already initialized |
| `init_if_needed` | create only if discriminator not present |
| `payer = <field>` | signer paying for rent |
| `space = <expr>` | byte size; use `T::SIZE` for fixed accounts |

### Validation

| Constraint | Effect |
|---|---|
| `has_one = <field>` | `self.<account>.<field> == self.<field>.address()` |
| `constraint = <expr>` | arbitrary bool — `ProgramError::ConstraintViolated` if false |

### Lifecycle

| Constraint | Effect |
|---|---|
| `close = <field>` | zero data, transfer lamports to `<field>`, reassign owner to system |

### SPL Token Init (quasar-spl)

| Constraint | Effect |
|---|---|
| `token::mint = <field>` | set mint when creating token account |
| `token::authority = <field>` | set authority when creating token account |

---

## Dynamic Fields — `String` and `Vec`

Use when the account needs variable-length data.

```rust
use quasar_lang::prelude::*;

// String<MAX_BYTES>
// Vec<ElementType, MAX_COUNT>

#[account(discriminator = 2)]
pub struct Profile {
    pub owner:    Address,
    pub score:    u64,
    pub name:     String<64>,            // up to 64 bytes
    pub bio:      String<512>,           // up to 512 bytes
    pub friends:  Vec<Address, 16>,      // up to 16 addresses
    pub scores:   Vec<PodU64, 8>,        // up to 8 u64s
}
```

Use the same account field style for fixed and dynamic account data types:
```rust
// Fixed struct
#[account(mut)]
pub escrow:  Account<Escrow>,

// Dynamic struct
pub profile: Account<Profile>,
```

`set_inner` for dynamic accounts also takes a payer account view and optional rent sysvar:
```rust
self.profile.set_inner(
    *self.signer.address(),       // owner: Address
    0u64,                         // score: u64
    "alice",                      // name: &str
    "web3 builder",               // bio: &str
    &[],                          // friends: &[Address]
    &[],                          // scores: &[PodU64]
    self.signer.to_account_view(), // payer for realloc
    Some(&**self.rent),            // rent sysvar for space calc
);
```

Auto-generated field accessors:
```rust
self.profile.owner     // Address — fixed fields accessed directly
self.profile.score     // u64
self.profile.name()    // &str   — dynamic fields via accessor method
self.profile.friends() // &[Address]
```

Individual field setters (for updates — handles realloc automatically):
```rust
self.profile.set_name("bob", self.signer.to_account_view(), Some(&**self.rent))?;
self.profile.set_friends(&new_friends, self.signer.to_account_view(), Some(&**self.rent))?;
```

---

## Sysvars

```rust
// In accounts struct:
pub rent:  Sysvar<Rent>,
pub clock: Sysvar<Clock>,
```

Usage in impl:
```rust
// Rent — compute minimum balance
let min = self.rent.minimum_balance(data_size);

// Clock — current slot and timestamp
let slot      = u64::from(self.clock.slot);
let timestamp = i64::from(self.clock.unix_timestamp);
let epoch     = u64::from(self.clock.epoch);
```

Off-chain sysvar access (when no account is passed):
```rust
let clock = Clock::get()?;
let rent  = Rent::get()?;
```

In QuasarSvm tests, sysvars are always available at their known addresses:
```rust
let rent  = quasar_svm::solana_sdk_ids::sysvar::rent::ID;
let clock = quasar_svm::solana_sdk_ids::sysvar::clock::ID;
```

---

## Remaining Accounts (`CtxWithRemaining`)

Use when the number of accounts is dynamic (e.g., multisig signers, batch operations).

### Program side

```rust
// Use CtxWithRemaining instead of Ctx:
#[instruction(discriminator = 0)]
pub fn create(ctx: CtxWithRemaining<Create>, threshold: u8) -> Result<(), ProgramError> {
    ctx.accounts.create_multisig(threshold, &ctx.bumps, ctx.remaining_accounts())
}
```

```rust
use quasar_lang::{prelude::*, remaining::RemainingAccounts};

impl Create {
    pub fn create_multisig(
        &mut self,
        threshold: u8,
        bumps: &CreateBumps,
        remaining: RemainingAccounts,
    ) -> Result<(), ProgramError> {
        let mut addrs = [Address::default(); 10];
        let mut count = 0usize;

        for account in remaining.iter() {
            let account = account?;
            if count >= 10 {
                return Err(ProgramError::InvalidArgument);
            }
            if !account.is_signer() {
                return Err(ProgramError::MissingRequiredSignature);
            }
            addrs[count] = *account.address();
            count += 1;
        }

        let signers = &addrs[..count];
        // use signers ...
        Ok(())
    }
}
```

`RemainingAccounts` iterator yields `Result<&AccountView, ProgramError>`. Each `AccountView` has:
- `.address()` → `&Address`
- `.is_signer()` → `bool`
- `.is_writable()` → `bool`
- `.lamports()` → `u64`
- `.data()` → `&[u8]`
- `.owner()` → `&Address`

### Test side — passing remaining accounts

```rust
use solana_instruction::AccountMeta;

let ix: Instruction = CreateInstruction {
    creator,
    config,
    rent,
    system_program,
    threshold,
    remaining_accounts: vec![
        AccountMeta::new_readonly(signer1, true),   // (address, is_signer)
        AccountMeta::new_readonly(signer2, true),
        AccountMeta::new_readonly(signer3, true),
    ],
}.into();
```

Add `solana-instruction` to dev-dependencies:
```toml
solana-instruction = { version = "3.2.0", features = ["bincode"] }
```

---

## Optional Accounts

Mark a field as optional with `Option<...>`:

```rust
pub referrer: Option<UncheckedAccount>,
```

In impl, check with `.is_some()` / `.map()` / `if let Some(r) = self.referrer`.

---

## Return Data

Set return data from an instruction (read by the caller or client):

```rust
use quasar_lang::return_data::set_return_data;

set_return_data(&bytes);   // &[u8]
```

---

## String Instruction Arguments

Pass variable-length strings as instruction arguments:

```rust
// In program handler:
#[instruction(discriminator = 2)]
pub fn set_label(ctx: Ctx<SetLabel>, label: String<32>) -> Result<(), ProgramError> {
    ctx.accounts.update_label(label)
}

// In impl, label is &str:
pub fn update_label(&mut self, label: &str) -> Result<(), ProgramError> {
    self.config.set_label(label, self.creator.to_account_view(), None)?;
    Ok(())
}
```
