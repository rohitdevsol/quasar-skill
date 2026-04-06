# Testing — Quasar Reference

Full reference for `QuasarSvm` — the in-process Solana VM for testing Quasar programs without a local validator.

---

## Setup

```toml
# Cargo.toml dev-dependencies
my_program-client = { path = "target/client/rust/my_program-client" }
quasar-svm        = { git = "https://github.com/blueshift-gg/quasar-svm" }
solana-address    = { version = "2.2.0", features = ["decode"] }
solana-pubkey     = { version = "4.1.0" }
solana-keypair    = "3.1.2"
```

Always run `quasar build` before testing — tests `include_bytes!` the compiled binary.

---

## Basic Structure

```rust
// src/tests.rs
extern crate std;   // re-enable std inside tests (cfg_attr suppresses it on-chain)

use quasar_svm::{Account, Instruction, Pubkey, QuasarSvm};
use solana_address::Address;
use my_program_client::MyInstruction;

fn setup() -> QuasarSvm {
    // include_bytes! path is relative to src/tests.rs
    let elf = include_bytes!("../target/deploy/my_program.so");
    QuasarSvm::new().with_program(&Pubkey::from(crate::ID), elf)
}

#[test]
fn test_something() {
    let mut svm = setup();
    // ...
}
```

For large test suites, use `std::fs::read` instead of `include_bytes!` to avoid recompiling on data changes:
```rust
let elf = std::fs::read("../../target/deploy/my_program.so").unwrap();
```

---

## Loading Programs

```rust
QuasarSvm::new()
    .with_program(&Pubkey::from(crate::ID), elf)    // your program
    .with_token_program()                             // SPL token program
    // chain more .with_program() calls for CPIs to other programs
```

---

## Account Helpers

```rust
// Funded system account (wallet)
quasar_svm::token::create_keyed_system_account(&pubkey, lamports)

// Empty account (for PDA placeholders)
Account {
    address: pda,
    lamports: 0,
    data: vec![],
    owner: quasar_svm::system_program::ID,   // or crate::ID if program-owned
    executable: false,
}

// Funded account owned by your program (for program-owned PDAs)
Account {
    address: vault,
    lamports: 1_000_000_000,
    data: vec![],
    owner: crate::ID,
    executable: false,
}
```

---

## Pre-loading Accounts

Use `svm.set_account()` when you need accounts to exist before the instruction runs. This is the right pattern for complex setup (many accounts, pre-existing state).

```rust
svm.set_account(Account {
    address: maker.pubkey(),
    lamports: 20_000_000_000,
    data: vec![],
    owner: quasar_svm::system_program::ID,
    executable: false,
});

// Or use helpers:
svm.set_account(quasar_svm::token::create_keyed_system_account(&user, 10_000_000_000));
svm.set_account(quasar_svm::token::create_keyed_mint_account(&mint_a, &MINT_SPEC));
svm.set_account(quasar_svm::token::create_keyed_associated_token_account(&owner, &mint, 0));
```

After `set_account`, those accounts don't need to be passed to `process_instruction`.

---

## Running Instructions

```rust
// Build the instruction from the auto-generated client struct
let ix: Instruction = DepositInstruction {
    signer: Address::from(user.to_bytes()),
    vault,
    system_program: Address::from(quasar_svm::system_program::ID.to_bytes()),
    amount: 1_000_000_000,
}.into();

// Pass accounts that are NOT already loaded via set_account
let result = svm.process_instruction(&ix, &[
    quasar_svm::token::create_keyed_system_account(&user, 10_000_000_000),
    Account { address: vault, lamports: 0, data: vec![], owner: crate::ID, executable: false },
]);
```

---

## Chaining Instructions

`result.accounts` contains the updated state of all touched accounts. Pass it to the next instruction:

```rust
let deposit_result = svm.process_instruction(&deposit_ix, &initial_accounts);
deposit_result.assert_success();

// next_accounts has the updated balances
let withdraw_result = svm.process_instruction(&withdraw_ix, &deposit_result.accounts);
withdraw_result.assert_success();
```

This simulates multiple instructions in the same test without resetting the VM state.

---

## Assertions

```rust
// Success
result.assert_success();

// Custom error (from #[error_code], starts at 3000)
result.assert_error(quasar_svm::ProgramError::Custom(3000));
result.assert_error(quasar_svm::ProgramError::Custom(3002));

// Runtime / native program errors
result.assert_error(quasar_svm::ProgramError::Runtime(String::from("IllegalOwner")));
result.assert_error(quasar_svm::ProgramError::Runtime(String::from("InvalidAccountData")));

// Standard program errors
result.assert_error(quasar_svm::ProgramError::MissingRequiredSignature);
result.assert_error(quasar_svm::ProgramError::InvalidArgument);

// Manual check with debug output
assert!(result.is_ok(), "instruction failed: {:?}", result.raw_result);
```

---

## Reading Results

```rust
// Get an account from the result
let vault_acct = result.account(&vault_pubkey).unwrap();
assert_eq!(vault_acct.lamports, 1_000_000_000);

// Compute units consumed
println!("CU: {}", result.compute_units_consumed);

// All updated accounts
let updated_accounts = result.accounts;
```

---

## Keypairs (for multi-signer tests)

```rust
use solana_keypair::{Keypair, Signer};

let maker = Keypair::new();
let taker = Keypair::new();

// Use .pubkey() for Pubkey, or convert to Address:
let maker_addr = Address::from(maker.pubkey().to_bytes());

// Pre-load as funded account:
svm.set_account(Account {
    address: maker.pubkey(),
    lamports: 20_000_000_000,
    data: vec![],
    owner: quasar_svm::system_program::ID,
    executable: false,
});
```

---

## Remaining Accounts in Tests

```rust
use solana_instruction::AccountMeta;  // needs solana-instruction in dev-deps

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

```toml
# dev-dependencies
solana-instruction = { version = "3.2.0", features = ["bincode"] }
```

---

## Common Test Patterns

### Setup function returning reusable state struct

```rust
struct State {
    maker: Keypair,
    vault: Address,
    mint: Pubkey,
}

fn setup() -> (QuasarSvm, State) {
    let elf = include_bytes!("../target/deploy/my_program.so");
    let mut svm = QuasarSvm::new()
        .with_program(&Pubkey::from(crate::ID), elf)
        .with_token_program();

    let maker = Keypair::new();
    svm.set_account(quasar_svm::token::create_keyed_system_account(&maker.pubkey(), 20_000_000_000));

    let (vault, _) = Address::find_program_address(
        &[b"vault", maker.pubkey().as_ref()],
        &crate::ID,
    );

    let state = State { maker, vault, mint: Pubkey::new_unique() };
    (svm, state)
}
```

### Verify negative case (attack scenario)

```rust
#[test]
fn test_unauthorized_withdraw() {
    let mut svm = setup();

    // First deposit as legitimate user
    let deposit_result = svm.process_instruction(&deposit_ix, &initial_accounts);
    deposit_result.assert_success();

    // Attacker tries to withdraw from someone else's vault
    let attacker = Pubkey::new_unique();
    let withdraw_ix: Instruction = WithdrawInstruction {
        signer: attacker,
        vault,
        amount: 1_000_000_000,
    }.into();

    let attack_result = svm.process_instruction(&withdraw_ix, &deposit_result.accounts);
    attack_result.assert_error(quasar_svm::ProgramError::Custom(3002));  // Unauthorized
}
```

### Pre-serialize account data (for testing state reads)

```rust
// Build an account with existing on-chain data using the client types:
use quasar_lang::client::{DynBytes, DynVec};
use my_program_client::MultisigConfig;

let config = MultisigConfig {
    creator: creator_addr,
    threshold: 2,
    bump: 255,
    label: DynBytes::new(b"my-multisig".to_vec()),
    signers: DynVec::new(vec![signer1, signer2, signer3]),
};

svm.set_account(Account {
    address: config_addr,
    lamports: 1_000_000,
    data: wincode::serialize(&config).unwrap(),
    owner: crate::ID,
    executable: false,
});
```

### Measure compute units

```rust
#[test]
fn test_cu_budget() {
    let (mut svm, state) = setup();
    let result = svm.process_instruction(&ix, &accounts);
    result.assert_success();
    println!("CU consumed: {}", result.compute_units_consumed);
    assert!(result.compute_units_consumed < 10_000, "too many CUs");
}
```
