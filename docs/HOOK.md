# The DARKHOOK hook

**One small program that makes any Socket pool private by rule.**

- Program: `GNRa1VQta9gMpBUfWHpe3nmSaScyDvNGu5KdprsXsr3p` (mainnet-beta, ~40 KB, no state)
- Source: [`programs/gate-hook/src/lib.rs`](../programs/gate-hook/src/lib.rs)
- Trusts exactly one private account: `35kTmDNFypu19FfRvJ2sWTwFySyBXvE6LC9Usz9uUwtF`
  (compiled in; see [Building](#building))

---

## 1. Hooks on Socket in one minute

[Socket](https://thesocket.app) is a concentrated-liquidity AMM on Solana where new
market logic ships as a **hook**, not as a new exchange. When a pool is created, its
creator fixes, forever:

- **one hook program** that Socket will call;
- a **permission mask** — which moments Socket calls it: before/after swap,
  before/after liquidity, and whether it may set the fee;
- a list of **extra accounts** the hook may read or write, listed literally.

Socket then calls the hook by cross-program invocation (CPI) at those moments, signed by
a pool-specific "hook authority" so nobody else can impersonate Socket. The hook answers
with a tiny reply (it may change the fee) — or returns an error, which **cancels the
whole transaction**.

## 2. What the DARKHOOK hook does

A DARKHOOK pool is created with:

| Setting | Value | Why |
|---|---|---|
| Hook program | `GNRa1VQt…sXsr3p` | the rule below |
| Mask | `1` (before swap only) | only swaps are checked; liquidity providers are never affected |
| Extra account | the **Instructions sysvar** (`Sysvar1nstructions1111111111111111111111111`, read-only) | lets the hook read every instruction of the transaction |

Before every swap the hook reads the transaction and applies three checks:

```
1. Is this a direct Socket swap on this very pool?        no → NotADirectSwap     (error 4)
2. Earlier in the transaction, did the private account
   pay the swapper (Withdraw → swapper or its token acct)? no → NotFundedFromVault  (error 5)
3. Later in the transaction, does the swapper deposit
   everything back into the private account (DepositAll)? no → NotSettledToVault   (error 6)
```

If all three hold, the hook replies "unchanged" and the swap runs at the pool's normal
price and fee. Solana transactions are atomic, so if the later deposit fails, the swap is
undone too — the hook can trust what it reads.

The core of the rule, verbatim from the source:

```rust
pub fn check_private(ixs: &[Ix], current: usize, pool: &Pubkey, shield: &Pubkey) -> Result<(), GateError> {
    let swap = ixs.get(current).ok_or(GateError::NotADirectSwap)?;
    if swap.program_id != SOCKET_PROGRAM_ID || swap.tag != Some(SOCKET_SWAP)
        || swap.accounts.len() <= SWAP_USER_B || &swap.accounts[SWAP_POOL] != pool
    {
        return Err(GateError::NotADirectSwap);
    }
    let swapper = swap.accounts[SWAP_SWAPPER];
    let ours = [swapper, swap.accounts[SWAP_USER_A], swap.accounts[SWAP_USER_B]];

    let funded = ixs[..current].iter().any(|ix| &ix.program_id == shield
        && ix.tag == Some(SHIELD_WITHDRAW)
        && ix.accounts.get(WITHDRAW_RECIPIENT).is_some_and(|r| ours.contains(r)));
    if !funded { return Err(GateError::NotFundedFromVault); }

    let settled = ixs[current + 1..].iter().any(|ix| &ix.program_id == shield
        && ix.tag == Some(SHIELD_DEPOSIT_ALL)
        && ix.accounts.get(DEPOSIT_DEPOSITOR) == Some(&swapper));
    if !settled { return Err(GateError::NotSettledToVault); }
    Ok(())
}
```

### What the hook guarantees

- **No public wallet can trade on the pool.** Every swap's money comes from the private
  account in the same transaction, and every result goes back into it.
- **No way around it.** The swap must be a top-level Socket instruction, so a wrapper
  program cannot hide what surrounds it; and the pool's hook can never be changed.
- **No custody, no switches.** The hook has no state, holds no funds, has no admin
  instruction and never changes the fee.

### What it does not do

- It does not hide **swap sizes** — Socket pools are public markets (hidden sizes are
  Phase B).
- It cannot stop someone from routing their *own* funds through a wallet they already
  revealed; that only de-anonymises themselves, never other users.

### Proven on mainnet

- A private swap passing through the hook:
  [`5j3y5NVW…7zJ6pbjq`](https://explorer.solana.com/tx/5j3y5NVWfVvHZUD1kQJxmbmhiLZ2gbS6eZvHpyfsyxPjG1Kn3ry5awxHtGnhVBWAM1NGHTmbW3K4ENAT7zJ6pbjq)
  — signed only by the relayer and a one-time wallet; the log line `socket:hook:0` is the
  hook being called.
- Refusals are covered by the end-to-end tests against the real Socket program
  (`tests/integration/private-e2e.mjs`): a plain swap → `NotFundedFromVault`; a funded
  swap that keeps its output → `NotSettledToVault`.

## 3. Launch your own private market

The hook is **permissionless**: anyone can create a Socket pool with it and earn the
pool's fees as a liquidity provider. You choose the fee tier, the starting price and
your liquidity range. Every such pool is private by the same rule.

**Today** a DARKHOOK pool must trade **SOL against USDC**, because the private account
holds SOL and USDC; private accounts for any token are the next step (Phase A+).

### With the script (Node ≥ 20, bash: Linux, macOS, WSL or Git Bash)

You need a Solana keypair file with some SOL (account rents + transaction fees), plus
the SOL and USDC you want to provide as liquidity.

```bash
git clone https://github.com/DarkHookPrivacy/darkhook.git && cd darkhook
(cd sdk && npm install && npm run build)
(cd tests/integration && npm install)

# dry run first: simulates everything, sends nothing
WALLET=~/my-keypair.json RELAYER=CLz6dudyf9x9zKRgzaogC3ZVxEjSFaD2Z5T63No2UmU6 \
PRICE=<USDC per SOL now> FEE_PPM=3000 NONCE=1 \
LIQ_SOL=1 LIQ_USDC=120 \
node tests/integration/private-setup.mjs

# then for real
SEND=1 WALLET=... (same variables) node tests/integration/private-setup.mjs
```

| Variable | Meaning |
|---|---|
| `WALLET` | your keypair file (JSON byte array); it pays and owns the pool and the position |
| `RELAYER` | the relayer that will serve private swaps on your pool. `CLz6dudy…UmU6` is the public DARKHOOK relayer; use your own address if you run one |
| `PRICE` | starting price in USDC per SOL — use the current market price |
| `FEE_PPM` | swap fee in parts per million (`3000` = 0.3 %, `10000` = 1 %) |
| `NONCE` | lets one wallet create several pools (1, 2, 3 …) |
| `LIQ_SOL`, `LIQ_USDC` | liquidity you add (both sides); `RANGE_TICKS` (default 7000) sets the range width |
| `RPC_URL` | optional, default `https://api.mainnet-beta.solana.com`; a dedicated RPC is faster |

It creates, from your wallet:

1. the Socket pool with the DARKHOOK hook, mask `1` and the Instructions-sysvar extra;
2. your liquidity position (both tokens, around `PRICE`, ±`RANGE_TICKS`);
3. the relayer's USDC token account (if missing) and an address lookup table for private
   swaps on your pool (it keeps the transaction under Solana's size limit).

Re-running is safe: steps that already exist are skipped. At the end it prints
`VITE_PRIVATE_POOL` and `VITE_PRIVATE_ALT` for your pool.

### Trading on your pool

The app in [`app/`](../app) serves one pool, set by `VITE_PRIVATE_POOL` and
`VITE_PRIVATE_ALT`. To offer private swaps on your market, deploy `app/` (for example on
Vercel, see [`app/README.md`](../app/README.md)) with those two values and your own
`RELAYER_SECRET_KEY` — the relayer you passed as `RELAYER` above. Private accounts are
shared by every DARKHOOK pool: SOL and USDC deposited through any DARKHOOK site can be
swapped on any DARKHOOK pool (notes move between sites with the app's backup / restore).

### By hand (any Socket client)

Create the pool with Socket's `Initialize` instruction and these hook settings:

```json
{
  "hookProgram": "GNRa1VQta9gMpBUfWHpe3nmSaScyDvNGu5KdprsXsr3p",
  "permissions": 1,
  "hookBudget": "0",
  "extras": [
    { "address": "Sysvar1nstructions1111111111111111111111111",
      "owner":   "Sysvar1111111111111111111111111111111111111",
      "writable": false }
  ]
}
```

Mints: wrapped SOL `So11111111111111111111111111111111111111112` and USDC
`EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`. The SDK builds the exact instruction
(`socketInitializeInstruction` in [`sdk/src/socket.ts`](../sdk/src/socket.ts)).

### Check that a pool is really private

A Socket pool account records the hook program it was created with, and Socket keeps
its extra accounts in the hook-list account `["extra-account-metas", pool]`; neither can
change after creation. `loadSocketPool(connection, pool)` in the SDK refuses a pool
whose account does not reference the DARKHOOK hook, and the app loads its pool through
it. On an explorer, a private swap on the pool shows the hook program
`GNRa1VQt…sXsr3p` invoked by Socket right before the swap.

## 4. Liquidity providers

- Adding and removing liquidity never calls the hook (mask `1`): LPs use normal Socket
  instructions from their own wallets.
- LPs earn the pool's swap fees like on any Socket pool.
- Price discovery comes from private traders (who must go through the private account),
  so a new pool should start at the market price and be seeded on both sides.

## Building

The hook trusts one private-account program, compiled in:

```bash
SHIELD_PROGRAM_ID=35kTmDNFypu19FfRvJ2sWTwFySyBXvE6LC9Usz9uUwtF \
  cargo build-sbf --manifest-path programs/gate-hook/Cargo.toml --sbf-out-dir target/deploy
sha256sum target/deploy/gate_hook.so   # 539e6e81…27d53de4, matches mainnet
```

Verify the deployed bytes with `tests/integration/verify-deploy.sh GNRa1VQta9gMpBUfWHpe3nmSaScyDvNGu5KdprsXsr3p target/deploy/gate_hook.so`.
