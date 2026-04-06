# quasar-skill
 
An agent skill for [Quasar](https://github.com/blueshift-gg/quasar) — the zero-copy Solana program framework written in Rust.
 
Install this skill to give your AI agent complete knowledge of Quasar's API: macros, account types, constraints, CPI patterns, `QuasarSvm` testing, and CLI commands.
 
---
 
## Install
 
```bash
npx skills add rohitdevsol/quasar-skill
```
 
Or install to a specific agent:
 
```bash
npx skills add rohitdevsol/quasar-skill -a claude-code
npx skills add rohitdevsol/quasar-skill -a cursor
npx skills add rohitdevsol/quasar-skill -a codex
```
 
---
 
## What's covered
 
- `#[program]` / `#[instruction(discriminator = N)]` — program entrypoint and dispatch
- `#[derive(Accounts)]` — account parsing with full constraint reference (`mut`, `signer`, `seeds`, `init`, `payer`, `space`, `address`)
- `#[account(discriminator = N)]` — zero-copy on-chain account types and `ZcT` companion structs
- `Ctx<T>` / `CtxWithRemaining<T>` — context types
- Account wrappers: `Account`, `Signer`, `UncheckedAccount`, `Program`, `SystemAccount`
- Pod integer types: `PodU64`, `PodI64`, etc.
- `require!`, `require_eq!`, `require_keys_eq!` constraint macros
- `#[event]` / `emit!` — on-chain events
- `#[error_code]` — typed program errors
- PDA derivation and `verify_program_address`
- CPI with system program, token program, PDA signer seeds
- `QuasarSvm` — in-process test runtime (no validator needed)
- CLI: `quasar init`, `quasar build`, `quasar test`, `quasar deploy`, `quasar new instruction`
- `Quasar.toml` config
- Full worked example (vault program)
- Comparison table: Quasar vs Anchor
 
---
 
## Compatibility
 
Works with Claude Code, Cursor, Codex, Cline, Windsurf, OpenCode, and any agent that supports the [Agent Skills spec](https://skills.sh).
 
---
 
## Repo structure
 
```
quasar-skill/
├── README.md
└── quasar/
    └── SKILL.md
```
 
---
 
## License
 
MIT
 
