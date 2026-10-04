# DARKHOOK Whitepaper

**The programmable privacy hook layer for onchain markets.**

Version 1.0 · October 2026 · Solana mainnet-beta

---

## Abstract

On a public blockchain every swap is signed by a wallet, so anyone can see who traded
what, when and for how much, and can follow that wallet everywhere else. Privacy tools
exist, but they are separate from the market: privacy is a detour the trader has to take,
and the market itself stays fully visible.

DARKHOOK turns privacy into a **rule of the market**. A DARKHOOK pool is a normal
[Socket](https://thesocket.app) pool on Solana with one extra program plugged in — the
**privacy hook**. Before every swap, Socket calls the hook, and the hook refuses the swap
unless the money came out of a shared zero-knowledge **private account** in the same
transaction and the whole result goes straight back into it. Nobody can trade on the pool
from a normal wallet, so every trade is made by a one-time wallet that is not linked to
any user.

This paper describes the design, the cryptography, what stays hidden and what does not,
the trust assumptions, and the roadmap. The launch is **unaudited software with deposit
caps**; Section 10 lists exactly what you are trusting.

---

## 1. The problem

A public ledger is a permanent, searchable record.

- **Trades are attributable.** A DEX swap names the wallet that made it. Wallet history
  reveals strategy, size, and holdings.
- **Fresh wallets do not stay fresh.** The moment a new wallet is funded from a known
  wallet, or sends its profits back, the two are linked for good.
- **Privacy is optional, so it is rare.** A trader who wants privacy has to leave the
  market, use a separate tool, and come back. Anyone who does not is fully exposed, and
  the few who do stand out.
- **The market cannot ask for privacy.** On a typical pool there is no way to say "only
  private trades here", so liquidity providers and traders cannot choose a private venue.

DARKHOOK addresses the last point directly: the market itself enforces privacy.

## 2. The idea: privacy as a hook

Socket is a concentrated-liquidity AMM where new market behaviour ships as a **hook** — a
program the pool calls at fixed moments — instead of as a new exchange. When a pool is
created, its creator fixes which hook program it uses, which moments call it (a permission
mask), and the literal extra accounts the hook can read. Socket calls the hook by
cross-program invocation, and a hook that returns an error cancels the whole transaction.

DARKHOOK uses exactly that mechanism:

> **A swap on a DARKHOOK pool is valid only if it is paid from the private account and
> settled back into the private account, in the same transaction.**

Because Solana transactions are atomic, the hook can read the transaction around the swap
and trust that what it sees will really happen, or nothing happens at all.

The product has three parts and two helpers:

| Part | What it is |
|---|---|
| **Private account** | A zero-knowledge shielded pool for SOL and USDC. Balances are secret *notes*; spending one is proved with a Groth16 proof. |
| **Market** | A normal Socket pool trading SOL against USDC, with liquidity providers who earn fees. |
| **Privacy hook** | A stateless program, plugged into the pool, that enforces the rule above before every swap. |
| *One-time wallet* | A throwaway key made in the browser for a single swap; it ends the transaction with nothing. |
| *Relayer* | A server that pays network fees and submits the transaction, so the user's wallet signs nothing. |

## 3. System overview

```
        your wallet                                   any wallet
            │ deposit  (public)                           ▲ withdraw (public)
            ▼                                             │
 ┌────────────────────────────────────────────────────────────────────┐
 │                  PRIVATE ACCOUNT  (zero-knowledge)                 │
 │        balances are secret notes · nobody sees whose they are      │
 └───────────────┬────────────────────────────────────▲───────────────┘
                 │ 1. pay the swap                     │ 3. take the result back
                 ▼                                     │
 ┌────────────────────────────────────────────────────────────────────┐
 │               SOCKET POOL  SOL / USDC  (public market)             │
 │   before every swap, Socket calls ──►  PRIVACY HOOK                │
 │                      "money from the private account?              │
 │                       result back into it?  no → refuse"           │
 └────────────────────────────────────────────────────────────────────┘
```

A user deposits (publicly), swaps any number of times (privately), and withdraws to any
wallet (publicly, with no link to the deposit). The swap in the middle is the part that
the hook makes private by rule.

## 4. The private account

The private account is a Solana program (`shield`) implementing a shielded pool for each
asset. It is built from standard components: Poseidon hashes, an incremental Merkle tree,
and Groth16 proofs verified with Solana's `alt_bn128` syscalls.

### 4.1 Notes

A **note** is a triple `(amount, nullifier, secret)`. The two random 31-byte values stay
in the user's browser. Only a fingerprint goes on chain:

```
inner         = Poseidon(nullifier, secret)
leaf          = Poseidon(inner, amount)
nullifierHash = Poseidon(nullifier)
```

All hashes are Poseidon over BN254 (circomlib-compatible parameters). Depositing inserts
the leaf; spending reveals the `nullifierHash` (never the leaf), which marks the note as
used without saying which leaf it came from.

### 4.2 The tree

Each asset has one pool (`["pool", mint]`; SOL uses the all-zero marker, SPL tokens are
held in `["vault", pool]`). A pool keeps an incremental Poseidon Merkle tree of depth 20
(1 048 576 leaves) and a ring of the last 30 roots, so a proof built against a recent
root stays valid while other users deposit.

### 4.3 Deposit

`Deposit { inner, amount }` moves `amount` into the pool and inserts
`Poseidon(inner, amount)`. A deposit is a normal, public transaction: it shows the
depositing wallet and the amount. `DepositAll { inner }` deposits the depositor's whole
balance — an amount only known at run time — and is what returns the output of a swap.
Both refuse a zero amount and a non-canonical `inner`.

### 4.4 Spend

To spend, the browser proves in zero knowledge (in about one to three seconds) that it
knows a note that is a leaf under a known `root`, and that it creates exactly the right
payouts. The circuit (`spend.circom`, 13 080 constraints, Groth16 on BN254) has the public
inputs, in order:

```
root, nullifierHash, changeLeaf, amount, fee, extDataHash
```

It proves knowledge of `(inAmount, nullifier, secret, path, changeNullifier,
changeSecret, changeAmount)` such that:

- `Poseidon(Poseidon(nullifier, secret), inAmount)` is a leaf under `root`;
- `nullifierHash = Poseidon(nullifier)`;
- `inAmount = amount + fee + changeAmount`, each of the three range-checked to 64 bits,
  so no wrap-around can create value;
- `changeLeaf` is a well-formed new note holding `changeAmount`.

`extDataHash` is the SHA-256 of `recipient ‖ feeRecipient ‖ amount ‖ fee` (with the top
byte cleared to fit the field). The proof therefore **binds the payout accounts and
amounts**: a relayer can submit a spend but cannot redirect the money or raise its fee.

A spend takes one note and leaves the rest as a **change note**, hidden like any other.

### 4.5 Withdraw checks

`Withdraw` verifies, in order: that `root`, `nullifierHash` and `changeLeaf` are canonical
field elements (a non-canonical alias would verify yet map to a different nullifier
account); that `root` is in the 30-root history; that the nullifier account
`["nullifier", pool, nullifierHash]` does not already exist; that the recipient and the fee
recipient are neither the pool nor the nullifier account; and that the Groth16 proof
verifies (about 100 000 compute units). Only then does it create the nullifier account,
insert the change leaf, and pay `amount` and `fee`.

### 4.6 Events and client state

The program logs a record for every insertion and spend, each naming its pool (a private
swap touches both pools in one transaction). Clients rebuild a pool's tree from its log
records, check the result against an on-chain root, and complete notes deposited with
`DepositAll` from the matching deposit record. The website therefore needs no database:
everything it knows comes from the chain and from the user's own browser.

## 5. The privacy hook

The hook (`gate-hook`) is a small stateless program (about 40 KB). It holds no funds, has
no admin instruction, and never changes the fee. A DARKHOOK pool is created with:

- **mask 1** — the hook is called before swaps only; adding or removing liquidity never
  calls it, so liquidity providers use ordinary Socket instructions;
- **one extra account** — the **Instructions sysvar**, read-only, which lets the hook read
  every top-level instruction of the transaction.

Before each swap the hook checks three things:

1. **A direct swap.** The instruction being executed is a top-level Socket `Swap` on this
   very pool. A wrapper program cannot hide what surrounds it. Otherwise: `NotADirectSwap`.
2. **Funded from the private account.** An earlier instruction is a private-account
   `Withdraw` whose recipient is the swapper or one of the swap's token accounts.
   Otherwise: `NotFundedFromVault`.
3. **Settled back into it.** A later instruction is a private-account `DepositAll` made
   by the swapper. Otherwise: `NotSettledToVault`.

If all three hold, the hook answers "unchanged" and the swap runs at the pool's normal
price and fee. The identity of the private account the hook trusts is compiled into it.

Two properties follow:

- **No public wallet can trade on the pool.** Every swap's input comes out of the private
  account and every output goes back into it. A wallet with a public history can only
  route its own funds through itself, which exposes nobody but that wallet.
- **The rule is not in the website.** It lives in the pool's hook, chosen when the pool is
  created. No front end, relayer or pool creator can switch it off for that pool. The hook
  program's own code, however, is upgradeable by its upgrade authority (Section 10).

## 6. A private swap, step by step

A private swap is **one** Solana transaction (version 0, with an address lookup table;
about 1 100 bytes and 300 000 compute units; the hook itself uses about 17 000). It is
signed by the relayer, which pays the fee, and by a one-time wallet `T` created in the
browser. The user's wallet does not appear.

```
create T's wSOL and USDC token accounts       (relayer pays the rent)
private account  Withdraw   note → T, fee → relayer, change note inserted
[SOL in]   transfer to T's wSOL account + SyncNative
Socket     Swap             T, exact-in, minimum-out from a simulated quote
                              └─ hook runs here and checks the transaction
[USDC out] private account  DepositAll from T's USDC account   → new note
[SOL out]  close T's wSOL account, return its rent, DepositAll from T
close T's token accounts                       (rents back to the relayer)
```

`T` ends with nothing, and the browser learns two new notes: the change and the output.

**Relayer safety.** The browser builds the exact message with the SDK and signs it with
`T`. The relayer rebuilds the same message from the request parameters, attaches `T`'s
signature and its own, and simulates it with signature checks before sending. It never
signs anything it did not build, and a transaction that fails simulation is never sent.
The relayer is repaid by the `fee` field of the proof, which the proof itself binds.

The relayer is a convenience, not a requirement of the programs: any wallet can submit a
spend that carries a valid proof, at the cost of appearing as its submitter.

## 7. What stays private, and what does not

DARKHOOK is honest about its limits.

| | |
|---|---|
| **Hidden** | **Who swaps.** Each swap is made by a one-time wallet paid from the private account. The proof shows "one of the notes is mine" without saying which. |
| **Hidden** | **Balances inside the account.** An amount is inside its note's hash; change notes hide what remains after a spend. |
| **Hidden** | **The link** between a deposit and the swaps and withdrawals it later pays for. |
| **Public** | **Money in and out.** A deposit shows the depositing wallet and amount; a withdrawal shows the receiving wallet and amount. |
| **Public** | **Swap sizes.** Socket pools are public markets: each private swap's input and output amounts are visible, only not whose they are. |
| **Public** | **Prices and liquidity** of the pool. |

### 7.1 Known leaks

- **Amount matching.** Depositing 2.3471 SOL and withdrawing exactly 2.3471 SOL a minute
  later can be matched by an observer. So can withdrawing the exact output of a swap. The
  app warns when a withdrawal equals a swap output. Round amounts, leaving a remainder
  inside, and waiting between steps all help.
- **Small anonymity set.** Privacy grows with the number of users and with time. With few
  users, timing and amounts narrow the possibilities. This is a property of every shielded
  pool, and it is the main limit at launch.
- **Relayer metadata.** The relayer sees request IP addresses and the receiving wallet of
  withdrawals. It never learns which deposit a spend consumes. A user can hide the IP with
  the usual tools.
- **Note custody.** Notes live in the user's browser. Whoever has them can spend the
  funds, and if they are lost the funds are lost. The app offers backup and restore.
- **Swap sizes are visible.** Hidden sizes are planned for Phase B (Section 11).

## 8. Markets and liquidity

**Anyone can launch a private market.** The hook is permissionless: any wallet can create
a Socket pool with the DARKHOOK hook, choose the fee tier and the starting price, add
liquidity, and earn the pool's swap fees. The SDK and a setup script do it in one command
(see [HOOK.md](./HOOK.md#3-launch-your-own-private-market)). Today such a pool must trade SOL against USDC, because
those are the assets the private account holds; private accounts for other tokens are
Phase A+.

**Liquidity providers** use ordinary Socket instructions from their own wallets — the hook
is never called for liquidity — and earn fees like on any Socket pool.

**Price behaviour.** In a concentrated-liquidity pool the price moves only when someone
swaps. On a DARKHOOK pool every swap is private, so the pool price follows the outside
market only as fast as private traders move it, and arbitrage must also go through the
private account. A new pool should therefore start at the market price and be seeded on
both sides, and liquidity providers should expect to track the price themselves. This is
the price paid for making the market private by rule.

**Fees.** The hook never changes the pool's fee. The only protocol-level charge is the
relayer's fee, set by whoever runs the relayer and bound into each proof.

## 9. Implementation

- **Programs:** `shield` (the private account) and `gate-hook` (the privacy hook), written
  in Rust on the plain `solana-program` crate; neither binary carries the project name.
- **Circuit:** `spend.circom`; trusted setup from Perpetual Powers of Tau (phase 1) plus
  one contribution and a public beacon (phase 2).
- **SDK:** `@darkhook/sdk` (TypeScript) builds notes, proofs, the private-swap transaction
  and the Socket instructions.
- **Website:** a static front end on Vercel with two serverless functions — an RPC proxy
  and the relayer. There is no database.
- **Testing:** unit tests for both programs (including a real Groth16 proof), SDK tests
  pinned to cross-language vectors, and end-to-end tests that run the real Socket program
  (cloned from mainnet) on a local validator. They check, among other things, that a plain
  swap is refused, that a funded swap which keeps its output is refused, private swaps in
  both directions, double spends, inflated amounts, fee raising, payout redirection and
  forged change notes.

### Mainnet deployment

| | |
|---|---|
| Private account (`shield`) | `35kTmDNFypu19FfRvJ2sWTwFySyBXvE6LC9Usz9uUwtF` |
| Privacy hook (`gate`) | `GNRa1VQta9gMpBUfWHpe3nmSaScyDvNGu5KdprsXsr3p` |
| Private SOL pool / USDC pool | `6UoPKozWtZZSQxXDZVcUD75RhLxCZoNLo1RAHoeJHhSy` / `CigJ7XHyKYcBZmh5uAzuL2b4Mz8k8rmgP2XzCDS5ENsb` |
| DARKHOOK Socket pool (SOL/USDC) | `Gc6zJAtkg36uCbk3SAvtE8NKzR9yjqeocETGe1G372fq` |
| Socket program | `24Qa8yMGZMuz3fqGAFPK36cu47d94o9iKv8GAKfbo54Q` |

A private swap on mainnet,
[`5j3y5NVW…7zJ6pbjq`](https://explorer.solana.com/tx/5j3y5NVWfVvHZUD1kQJxmbmhiLZ2gbS6eZvHpyfsyxPjG1Kn3ry5awxHtGnhVBWAM1NGHTmbW3K4ENAT7zJ6pbjq),
has exactly two signers — the relayer and a one-time wallet — and its log shows Socket
calling the hook (`socket:hook:0`) before the swap.

## 10. Security and trust

DARKHOOK is **unaudited** and it **custodies funds**. This is exactly what you trust:

| Component | What you trust | Mitigation |
|---|---|---|
| **Programs** | That the code is correct. Both are upgradeable by one upgrade authority, so that authority could in principle change them. | Internal review and tests only so far. An external audit, then a multisig or a freeze, is planned. The hook can be rebuilt from source and compared byte for byte with the deployed program. |
| **Deposit caps** | — | A launch cap on each pool (deposits minus withdrawals) bounds the funds at risk; it never blocks withdrawals. Set by the upgrade authority: 50 SOL and 5 000 USDC at launch. |
| **Trusted setup** | That at least one phase-2 contributor discarded their secret. | Phase 1 is Perpetual Powers of Tau. Phase 2 has one contribution plus a public beacon, which is a weak point; more contributions will be added before caps are raised. A forged proof would need every contributor's secret. |
| **Relayer** | To submit your transaction and not censor it; it sees your IP and the withdrawal recipient. | It cannot steal or redirect (the proof binds payouts), cannot learn which deposit is spent, and anyone can submit a proof themselves. |
| **Your browser** | Holds your notes. | Backup and restore; losing the notes loses the funds. |
| **Socket** | That Socket's program behaves as documented. | DARKHOOK builds on its public hook interface and tests against the mainnet binary. |

The hook moves no funds and holds no state: its only power is to refuse a swap.

## 11. Roadmap

Planned work, in order, without dates:

- **Phase A (live).** Private account for SOL and USDC; privacy hook; relayer; launch
  caps. A permissionless hook: anyone can launch a private SOL/USDC Socket market.
- **Phase A+.** Private accounts for any SPL token, so any pair can have a private market;
  an in-app "launch a private market" flow and a pool list; a Socket allowlist for routing.
- **Phase B.** Hidden swap sizes (sealed batches inside the private account, with view
  keys); an external audit; a multisig for the upgrade authority; higher caps; optional
  association sets, which let a user prove their funds are clean without revealing which
  deposit is theirs.



## References

- Socket, a concentrated-liquidity AMM with hooks for Solana: <https://thesocket.app>
- J. Groth, *On the Size of Pairing-based Non-interactive Arguments* (Groth16), 2016:
  <https://eprint.iacr.org/2016/260>
- L. Grassi, D. Khovratovich, C. Rechberger, A. Roy, M. Schofnegger, *Poseidon: A New Hash
  Function for Zero-Knowledge Proof Systems*, 2019: <https://eprint.iacr.org/2019/458>
- circom, a circuit compiler: <https://docs.circom.io> · snarkjs, a Groth16 prover and
  verifier: <https://github.com/iden3/snarkjs>
- Perpetual Powers of Tau, the phase-1 trusted setup used by the circuit
  (Privacy and Scaling Explorations)

*DARKHOOK — launch a market, hook in privacy, let it evolve privately.*
