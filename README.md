# AMM

A constant-product automated market maker (AMM) built with Anchor and Rust on Solana. The program supports pool initialization, liquidity provision, liquidity withdrawal, and token swaps with configurable fees collected in treasury accounts.

## What the Program Does

The AMM manages two tokens:

- **Token X**: the first pool asset.
- **Token Y**: the second pool asset.
- **LP token**: represents a user's share of the pool's liquidity.

Each pool stores its configuration in a PDA and holds its reserves in token accounts owned by that PDA. Swap fees are kept separate from the pool reserves in treasury token accounts.

## Project Structure

```text
programs/amm-video/
  src/
    lib.rs                 Program entrypoints
    state.rs               Pool configuration account
    error.rs               Program errors
    instructions/
      initialize.rs        Pool and account creation
      deposit.rs           Add liquidity and mint LP tokens
      withdraw.rs          Remove liquidity and burn LP tokens
      swap.rs              Exchange Token X and Token Y
  tests/
    tests.rs               LiteSVM integration tests
    ix_handlers/           Test instruction builders
```

## Requirements

Install the following tools before running the project:

- Rust and the toolchain specified in `rust-toolchain.toml`
- Solana CLI
- Anchor CLI
- Cargo

The project uses Anchor `1.0.1` dependencies. The installed Anchor CLI should be compatible with the project dependencies.

## Build the Program

From the repository root, run:

```bash
anchor build
```

This compiles the program and generates the deployable shared object at:

```text
target/deploy/amm_video.so
```

The integration tests load this file into LiteSVM, so it must exist before running the tests.

## Program Instructions

The program exposes four instructions from `src/lib.rs`:

```rust
initialize(seed, fee, authority)
deposit(amount, max_x, max_y)
withdraw(amount, min_x, min_y)
swap(is_x, amount_in, min_amount_out)
```

The instruction arguments use token base units. For example, with six-decimal tokens, `1_000_000` represents one token.

### 1. Initialize a Pool

`initialize` creates the accounts required by a new pool.

#### Inputs

- `seed`: unique value used to derive the pool configuration PDA.
- `fee`: swap fee in basis points. `100` means `1%`; the maximum is `10,000` basis points.
- `authority`: optional authority stored in the pool configuration.

#### Accounts Created

1. **Config PDA**
   - Seeds: `config`, `seed`
   - Stores the token mints, fee, lock state, and PDA bumps.
2. **LP mint PDA**
   - Seeds: `lp`, `config`
   - Mint authority: the config PDA.
   - Decimals: six.
3. **Pool vaults**
   - `vault_x` stores Token X reserves.
   - `vault_y` stores Token Y reserves.
   - Both vaults are associated token accounts owned by the config PDA.
4. **Treasury PDA**
   - Seeds: `treasury`, `config`
   - Used as the authority for fee accounts.
5. **Treasury token accounts**
   - `treasury_x` stores Token X swap fees.
   - `treasury_y` stores Token Y swap fees.

Initialization starts the pool unlocked. A fee greater than `10,000` basis points is rejected.

### 2. Deposit Liquidity

`deposit` lets a user add both assets to the pool and receive LP tokens.

1. The program checks that the pool is not locked.
2. The requested LP token amount must be greater than zero.
3. For the first deposit, the user supplies `max_x` and `max_y` as the initial amounts.
4. For later deposits, the program calculates proportional Token X and Token Y amounts from the current reserves and LP supply.
5. The calculated amounts must be within `max_x` and `max_y`; otherwise the transaction fails with a slippage error.
6. Token X and Token Y move from the user's accounts into the pool vaults.
7. The config PDA signs a CPI that mints the requested LP tokens to the user.

This keeps the pool's liquidity ratio aligned with its existing reserves after the initial deposit.

### 3. Withdraw Liquidity

`withdraw` lets a user redeem LP tokens for a proportional share of the reserves.

1. The program checks that the pool is not locked.
2. The requested LP amount must be greater than zero.
3. Token X and Token Y redemption amounts are calculated from the user's LP amount, total LP supply, and current vault balances.
4. The calculated amounts must meet `min_x` and `min_y`; otherwise the transaction fails with a slippage error.
5. The user's LP tokens are burned.
6. The config PDA signs CPIs transferring the redeemed assets from the pool vaults to the user.

### 4. Swap Tokens

`swap` exchanges one pool asset for the other using the constant-product curve.

- `is_x = true`: the user swaps Token X for Token Y.
- `is_x = false`: the user swaps Token Y for Token X.
- `amount_in`: gross amount supplied by the user.
- `min_amount_out`: minimum amount the user accepts.

The swap flow is:

1. The program checks that the pool is unlocked and `amount_in` is not zero.
2. It initializes the constant-product curve using the current vault balances, LP supply, configured fee, and six-decimal precision.
3. The curve calculates the output amount and the fee.
4. The output must meet `min_amount_out`.
5. The net input, calculated as `amount_in - fee`, is transferred to the corresponding pool vault.
6. The fee is transferred to the matching treasury account:
   - Token X input fee -> `treasury_x`
   - Token Y input fee -> `treasury_y`
7. The config PDA signs a CPI transferring the output token from the opposite pool vault to the user.

Keeping fees outside the reserve vaults makes the treasury balance explicit and prevents fees from being counted as pool liquidity by the swap curve.

## Account Relationships

```text
Config PDA
  |-- owns vault_x (Token X reserves)
  |-- owns vault_y (Token Y reserves)
  |-- controls mint_lp (LP token mint)
  |
  `-- references Treasury PDA
        |-- owns treasury_x (Token X fees)
        `-- owns treasury_y (Token Y fees)
```

The user's associated token accounts hold their Token X, Token Y, and LP token balances. Token transfers use the SPL Token program, while associated token accounts are created through the Associated Token program.

## Testing

The project uses LiteSVM for fast in-process integration tests. The test setup creates two six-decimal mints, initializes a pool, creates user token accounts, and executes each instruction.

Run the complete test suite with:

```bash
anchor build
cargo test -p amm-video --test tests
```

The test suite covers:

- Pool initialization
- Initial liquidity deposit
- Liquidity withdrawal
- Token swap

## Local Development Workflow

1. Install the required Rust, Solana, and Anchor tooling.
2. Review the program ID in `Anchor.toml` and `src/lib.rs`.
3. Build the program with `anchor build`.
4. Run the LiteSVM tests with `cargo test -p amm-video --test tests`.
5. Make changes under `programs/amm-video/src`.
6. Update instruction builders and integration tests when account layouts or instruction arguments change.
7. Format Rust code with:

   ```bash
   cargo fmt --all
   ```

8. Rebuild and rerun the tests before submitting changes.

## Important Implementation

- Swap fees are configured in basis points and are limited to `10,000`.
- Pool and LP mint PDA addresses depend on the pool seed and config address.
- Pool vaults and treasury accounts use the standard SPL Token program and associated token account derivation.
- The LP mint is currently created with six decimals.
- The current implementation provides treasury fee accounts but does not include a separate instruction for withdrawing treasury funds.
