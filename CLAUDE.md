# CLAUDE.md

> AI agent context file. Loaded automatically at the start of every Claude Code session.
> This is the project's "operating manual" for AI agents.

## Project Overview

Solana protocol built with Anchor framework in a monorepo structure.

**Tech Stack:**
- On-chain: Rust + Anchor 0.32.1
- SDK: TypeScript (ES Module)
- Frontend: Next.js 14 + Solana Wallet Adapter
- Keeper: Deno (off-chain automation)
- Tests: Jest + ts-jest

**Monorepo Layout:**
- `programs/protocol/` — On-chain program
- `sdk/` — TypeScript SDK (@solana-boilerplate/sdk)
- `app/` — Next.js frontend
- `keeper/` — Deno keeper bot
- `tests/` — Integration tests

## Core Documentation

- `docs/SPECIFICATION.md` — Instruction interfaces, parameters, validations
- `docs/ARCHITECTURE.md` — Account model, PDA seeds, module dependencies

**Always read SPECIFICATION.md before implementing an instruction.**

## Development Commands

```bash
# Build
anchor build
./scripts/build.sh devnet      # devnet features enabled
./scripts/build.sh mainnet     # production build

# Test
pnpm test
pnpm test -- --testPathPattern="my_test.test.ts"  # specific test

# Check test results (structured JSON output)
cat test_result.json | jq '.numPassedTests, .numFailedTests'
cat test_result.json | jq '.testResults[].assertionResults[] | select(.status == "failed")'

# SDK
cd sdk && pnpm build
cd sdk && pnpm dev             # watch mode

# Keeper
cd keeper && deno task dev

# Program logs (after tests)
grep "consumed" .anchor/program-logs/*.log
```

## ⚠️ Critical Rules

- **DO NOT run tests directly.** Tests take 3+ minutes (validator start → deploy → Jest). Ask the user to run them, then read `test_result.json` for results.
- **DO NOT use `anchor test` directly.** Use `pnpm test` instead.
  - **Why:** `pnpm test` runs the configured `package.json "test"` script, which includes `--features` flags (e.g., `anchor test -- --features devnet,test-feature`). Running bare `anchor test` skips those flags → program compiles with wrong settings → tests fail in hard-to-diagnose ways.
  - **Rule:** If you add `[features]` to `Cargo.toml`, update `package.json "test"` to pass them: `"anchor test -- --features your-feature"`.
- **DO NOT commit `.env` files.** They contain private keys.
- **DO NOT modify multiple instructions in one change.** One instruction per change, test, then move on.
- **ALWAYS use checked arithmetic** (`checked_add`, `checked_sub`, `checked_mul`, `checked_div`) for all on-chain math.
- **ALWAYS update this file** when changing architecture, PDA seeds, error codes, or test structure.

## Architecture

### Program Module Structure

```
programs/protocol/src/
├── lib.rs             # Entry points (delegates to instruction handlers)
├── constants.rs       # PDA seeds, protocol parameters
├── error.rs           # All error codes
├── instructions/      # One file per instruction
│   └── mod.rs         # pub mod + pub use
├── state/             # Account structures
│   └── mod.rs
└── math/              # Business logic
    ├── mod.rs
    └── wad.rs         # Fixed-point WAD arithmetic (10^18)
```

### Instruction Handler Pattern (3-Phase)

Every instruction handler follows this pattern:

```rust
pub fn process_my_instruction(ctx: Context<MyInstruction>, args...) -> Result<()> {
    // Phase 1: Validation & Calculation (immutable borrow scope)
    let result = {
        let state = &ctx.accounts.my_state;
        require!(condition, ErrorCode::SomeError);
        // All reads and math here
        calculated_value
    }; // borrow ends here

    // Phase 2: CPIs (token transfers, mints, burns)
    let signer_seeds = &[&[b"seed", key.as_ref(), &[bump]][..]];
    token::transfer(cpi_ctx.with_signer(signer_seeds), amount)?;

    // Phase 3: State update (mutable borrow)
    let state = &mut ctx.accounts.my_state;
    state.value = result;
    Ok(())
}
```

### Solana Account Model (Key Concept)

Unlike EVM where contract = logic + state, Solana separates them:
- **Program** = logic only (read-only, executable)
- **Account** = state only (data storage, owned by a program)

When defining instructions, explicitly specify which accounts are read vs written.

## PDA Seeds

<!-- Add your PDAs here. Keep this table updated!

| PDA Name | Seeds | Bump Stored? | Purpose |
|----------|-------|-------------|---------|

-->

_No PDAs defined yet. Add them as you implement instructions._

## Important Implementation Notes

### On-Chain Math: Two Approaches

This boilerplate provides **two math approaches**. Choose based on your protocol:

#### 1. u128 Integer-Ratio Math (for AMM proportional calculations)

For proportional calculations like `amount * supply / reserve`, `fee * amount / 10000`,
or any integer-ratio math, use u128 intermediates with `checked_*` operations:

```rust
// Proportional: u128 intermediates
let lp_tokens = (amount_a as u128)
    .checked_mul(lp_supply as u128)?
    .checked_div(reserve_a as u128)? as u64;

// Ceil division for fees
let fee = ((amount_in as u128)
    .checked_mul(fee_rate_bps as u128)?
    .checked_add(9999)?
    .checked_div(10000)?) as u64;
```

#### 2. WAD Fixed-Point Math (for rate-based protocols)

`math/wad.rs` provides 10^18 fixed-point arithmetic for protocols needing fractional precision
(compound interest rates, price oracles, lending protocols):

```rust
use crate::math::wad::*;

let wad_value = to_wad(lamports)?;           // u64 → u128 WAD
let result = multiply_wad(a_wad, b_wad)?;     // WAD × WAD → WAD
let lamports = from_wad_floor(wad_value)?;    // WAD → u64 (round down)
let lamports = from_wad_ceil(wad_value)?;     // WAD → u64 (round up)
```

> **Which to use?** Check `docs/SPECIFICATION.md` — it specifies the math approach for this project.
> AMM (constant product) → u128 integer-ratio. Lending/oracle → WAD.

### Rounding Rules (Protocol Safety)

| Situation | Direction | Reason |
|-----------|-----------|--------|
| User **pays** | ceil (round up) | Protocol receives more |
| User **receives** | floor (round down) | Protocol pays less |
| Fee calculation | ceil (round up) | Protocol collects at least minimum |
| LP token issuance | floor (round down) | User receives less |

### Compute Unit (CU) Budget

- Default: 200,000 CU per instruction
- Check after implementing: `grep "consumed" .anchor/program-logs/*.log`
- If over budget: split into multiple instructions or optimize math

## Testing

### Seed ID Ranges

To prevent PDA collisions in parallel tests, each test file uses a unique seed range:

<!-- Add your test files here:

| Test File | Seed Range | Description |
|-----------|------------|-------------|
| my_feature.test.ts | 100-199 | Feature tests |

-->

_No test files yet. Assign ranges as you add tests._

### Test Helper Functions

- `setupTestContext(seedId)` — Creates provider, program, funded payer
- `getOrCreateATA(provider, mint, owner, payer)` — Gets or creates Associated Token Account

### Debugging Failed Tests

1. Run tests: `pnpm test`
2. Check results: `cat test_result.json | jq '.testResults[].assertionResults[] | select(.status == "failed")'`
3. Fix based on structured error messages
4. Repeat

## Error Codes

| Code | Name | Description |
|------|------|-------------|
| 6000 | MathOverflow | Arithmetic overflow |
| 6001 | MathError | Division by zero or invalid operation |
| 6002 | MathUnderflow | Arithmetic underflow |

_Add error codes as you implement instructions._

## Security Checklist

### AI Can Detect
- ✅ Missing `checked_*` arithmetic
- ✅ Missing access control (Signer verification)
- ✅ Missing input validation (amount > 0, valid ranges)
- ✅ PDA seed/bump mismatches
- ✅ Missing account ownership checks

### Human Must Verify
- ⚠️ Economic attack vectors (sandwich attacks, oracle manipulation)
- ⚠️ Multi-instruction state manipulation
- ⚠️ Flash loan vulnerabilities
- ⚠️ MEV (Miner Extractable Value) exposure
- ⚠️ Business logic correctness (does the math make economic sense?)

### Anchor Feature Flags (Cargo.toml)

**`init-if-needed` (Currently enabled)**
- ⚠️ **NOT automatically dangerous** — only risky if misused in code
- Safe usage: One-time account initialization (config, registry)
- Unsafe usage: User-specific accounts (wallets, positions) → re-initialization attacks
- Rule: Use `#[account(init)]` for user accounts, reserve `init_if_needed` for global singletons

**Why it's in workspace dependencies:**
```toml
[workspace.dependencies]
anchor-lang = { version = "0.32.1", features = ["init-if-needed"] }
```
- Available for use, but NOT active until `#[account(init_if_needed)]` is explicitly added
- AI should flag any `init_if_needed` usage in instruction code for human review

### Known False Positives
These are intentional for a boilerplate/development setup:
- ✅ Placeholder program ID (`11111...1`) — See "Deployment Checklist" below
- ✅ Broad dependency versions (Next.js `^14.0.0`) — Lock before production
- ✅ Missing `pnpm-lock.yaml` — Run `pnpm install` to generate
- ✅ Keeper with `-A` flag — Acceptable for trusted off-chain automation

## Deployment Checklist

Before deploying to devnet/mainnet, verify:

### 1. Program ID (CRITICAL)
```bash
# Current state (placeholder):
declare_id!("11111111111111111111111111111111");

# Steps to fix:
anchor build                                    # Generate keypair
solana address -k target/deploy/protocol-keypair.json
# Copy address to lib.rs declare_id!()
# OR use vanity address:
solana-keygen grind --starts-with ABC:1        # Custom prefix
```

### 2. Dependency Locking
```bash
# Lock versions for reproducible builds
pnpm install          # Generates pnpm-lock.yaml (commit this!)
cd app && pnpm install
cd sdk && pnpm install
```

### 3. Build Configuration
```bash
# Verify Anchor.toml [programs.localnet] has correct program ID
anchor build --verifiable                       # Mainnet builds
./scripts/build.sh mainnet                      # Our custom build script
```

### 4. Security Audit
- [ ] No `init_if_needed` on user-specific accounts
- [ ] All math uses `checked_*` operations
- [ ] All Signer constraints in place
- [ ] No hardcoded private keys in code
- [ ] `.env` files in `.gitignore`
- [ ] Error messages don't leak sensitive info

### 5. Keeper Permissions
```bash
# Review deno.json for production:
# --allow-net=<specific-domains>  (not --allow-net)
# --allow-read=<specific-paths>   (not -A)
# --allow-write=<specific-paths>
```

### 6. Frontend Configuration
```bash
# app/.env.production (create before deploy)
NEXT_PUBLIC_CLUSTER=mainnet-beta
NEXT_PUBLIC_PROGRAM_ID=<your-deployed-program-id>
# NEVER commit .env files with private keys
```

## AI Collaboration Guide

### Effective Prompt Structure (5 Elements)

```
[1] What: Implement the deposit instruction
[2] Where: programs/protocol/src/instructions/deposit.rs
[3] How: Follow the 3-phase pattern in initialize.rs
[4] Constraints: Use checked_* math, ceil rounding for fees, CU < 200k
[5] Verification: Write deposit.test.ts, seed range 200-299
```

### Incremental Build Order

```
Step 1: Happy path only (no error handling)
Step 2: Add validations and error codes
Step 3: Add edge case tests
Step 4: Optimize CU if needed
```

### Adding a New Instruction (Checklist)

1. Define the instruction in `docs/SPECIFICATION.md`
2. Create `instructions/my_instruction.rs` (3-phase pattern)
3. Add accounts struct with Anchor validation attributes
4. Add to `instructions/mod.rs` (pub mod + pub use)
5. Add entry point to `lib.rs`
6. Add error codes to `error.rs`
7. Update this file: Architecture, PDA Seeds, Error Codes
8. Write tests with assigned seed range
9. Create SDK instruction builder in `sdk/src/instructions/`

### Session Handoff

Before ending a session, ask AI:
> "Summarize progress and remaining work in CLAUDE.md"

This ensures the next session picks up where you left off.
