# DARKHOOK Protocol

**The programmable privacy hook layer for onchain markets.**

DARKHOOK makes a [Socket](https://thesocket.app) market private *by rule*. It has two
on-chain programs:

- **the private account** (`programs/shield`) — a zero-knowledge shielded pool where SOL
  and USDC live as secret notes with hidden amounts;
- **the privacy hook** (`programs/gate-hook`) — a Socket hook that refuses every swap
  that is not funded from the private account and settled back into it.

A pool created with the hook cannot be traded any other way.

---

## 1. What is hidden, what is not

| | |
|---|---|
| **Hidden** | *Who swaps.* Each swap is signed by a one-time wallet paid from the private account; a Groth16 proof shows "one of the notes is mine" without saying which. The user's wallet does not appear in the swap transaction. |
| **Hidden** | *Balances inside the account.* A note's amount is inside its hash; change notes hide what is left after a spend. |
| **Public** | *Money in and out.* A deposit shows the depositing wallet and the amount; a withdrawal shows the receiving wallet and the amount. |
| **Public** | *Swap sizes.* Socket pools are public CLMMs: each private swap's input and output amounts are visible (not whose they are). |
| **Not a guarantee** | *Anonymity set.* Privacy grows with the number of users and with time between steps. Depositing 2.3471 SOL and withdrawing exactly 2.3471 SOL, or withdrawing the exact output of a swap, can be matched by amount. |

## 2. The private account (`programs/shield`)

### Notes and tree

A note is `(amount, nullifier, secret)` with random 31-byte `nullifier`, `secret`.

```
inner = Poseidon(nullifier, secret)
leaf  = Poseidon(inner, amount)            // BN254, x^5, circomlib-compatible
nullifierHash = Poseidon(nullifier)
```

One pool per asset (`["pool", mint]`; SOL uses the all-zero mint and holds lamports,
SPL pools hold tokens in `["vault", pool]`). Each pool keeps an incremental Poseidon
Merkle tree of depth 20 (1 048 576 leaves) computed with the `sol_poseidon` syscall,
and a ring of the last 30 roots.

### Instructions

| # | Instruction | Effect |
|---|---|---|
| 0 | `InitPool { cap }` | Upgrade authority only. Creates the pool (and SPL vault). |
| 1 | `Deposit { inner, amount }` | Moves `amount` in; inserts `Poseidon(inner, amount)`. |
| 2 | `Withdraw { proof, root, nullifierHash, changeLeaf, amount, fee }` | Spends one note: pays `amount` to the recipient and `fee` to the fee recipient, burns the nullifier, inserts the change note. |
| 3 | `SetCap { cap }` | Upgrade authority only. |
| 4 | `DepositAll { inner }` | Deposits the depositor's whole balance (lamports, or its token account), an amount known only at run time — the output of a swap. |

`Deposit` / `DepositAll` refuse a zero amount, a zero or non-canonical `inner`, and any
deposit that would push `balance` (deposits minus withdrawals) above `cap`. The cap
never blocks withdrawals.

### The spend circuit (`circuits/spend.circom`)

13 080 constraints, Groth16 over BN254. Public inputs, in order:
`root, nullifierHash, changeLeaf, amount, fee, extDataHash`.

It proves knowledge of `(inAmount, nullifier, secret, path, changeNullifier,
changeSecret, changeAmount)` such that

- `Poseidon(Poseidon(nullifier, secret), inAmount)` is a leaf under `root`;
- `nullifierHash = Poseidon(nullifier)`;
- `inAmount = amount + fee + changeAmount`, each of the three range-checked to 64 bits
  (no wrap-around can mint value);
- `changeLeaf = Poseidon(Poseidon(changeNullifier, changeSecret), changeAmount)`.

`extDataHash = sha256(recipient ‖ feeRecipient ‖ amount_le ‖ fee_le)` with the top byte
cleared binds the payout accounts; a relayer cannot redirect funds or raise its fee.

### Withdraw checks, in order

1. `root`, `nullifierHash`, `changeLeaf` are canonical scalars (< r): a non-canonical
   alias would verify yet map to a different nullifier PDA;
2. `root` is in the 30-root history;
3. the nullifier PDA `["nullifier", pool, nullifierHash]` is not owned by the program;
4. recipient and fee recipient are neither the pool nor the nullifier account;
5. the Groth16 proof verifies (alt_bn128 syscalls, ~100k CU);
6. the nullifier PDA is created (tolerating lamports pre-sent to its address), the
   change leaf is inserted, and `amount` / `fee` are paid.

### Events (`sol_log_data`)

Every record names its pool, because a private swap touches both pools in one
transaction:

```
LEAF     [0, pool, index u32 le, leaf, new root]     every insertion
WITHDRAW [1, pool, nullifierHash]
DEPOSIT  [2, pool, index u32 le, amount u64 le]      the amount behind a deposited leaf
```

Clients rebuild a pool's tree from its LEAF records (attributed to this program by
invoke depth), check it against an on-chain root, and complete deposit-all notes from
the DEPOSIT record of their transaction.

### Errors (`ProgramError::Custom(n)`)

0 InvalidInstruction · 1 NotAuthority · 2 InvalidPoolAccount · 3 InvalidVault ·
4 InvalidMint · 5 CapExceeded · 6 TreeFull · 7 InvalidCommitment · 8 UnknownRoot ·
9 NullifierSpent · 10 InvalidNullifierAccount · 11 InvalidProof · 12 FeeTooHigh ·
13 InvalidFieldElement · 14 InvalidAccounts · 15 InsufficientPoolBalance ·
16 AlreadyInitialized · 17 InvalidAmount · 18 MissingSignature.

## 3. The privacy hook (`programs/gate-hook`)

Socket lets each pool attach one hook program, fixed at creation together with a
permission mask and a list of literal extra accounts. A DARKHOOK pool uses:

- **mask 1** — before-swap only (liquidity never calls the hook, LPs are unaffected);
- **one extra** — the Instructions sysvar (`Sysvar1nstructions…`, owner `Sysvar1111…`,
  read-only), so the hook can read every top-level instruction of the transaction.

The hook has no state, moves no funds and returns no fee. Before each swap it checks:

1. the swap is a **direct, top-level Socket `Swap` on this pool** (a wrapper program
   cannot hide what surrounds it) — else `NotADirectSwap` (4);
2. **funded from the vault** — an earlier instruction is a shield `Withdraw` whose
   recipient is the swapper or one of the swap's token accounts — else
   `NotFundedFromVault` (5);
3. **settled into the vault** — a later instruction is a shield `DepositAll` by the
   swapper — else `NotSettledToVault` (6).

Transactions are atomic, so those instructions really execute. The trusted shield id
is compiled into the hook (`SHIELD_PROGRAM_ID` at build time). A wallet with a public
history cannot trade on such a pool; at most it could route its own deposited funds
through itself, which only de-anonymises itself.

## 4. A private swap

One v0 transaction (an address lookup table holds the fixed accounts; ~1 120 bytes,
~300k CU), signed by the relayer (fee payer) and a throwaway wallet `T` created in
the browser:

```
create T's wSOL and USDC token accounts         (relayer pays rent)
shield.Withdraw  note → T (or T's USDC account), fee → relayer, change note inserted
[SOL in]  transfer to T's wSOL account + SyncNative
Socket.Swap      T, exact-in, min-out from a simulated quote
[USDC out] shield.DepositAll from T's USDC account          → new note
[SOL out]  close T's wSOL account to T, return its rent, shield.DepositAll from T
close T's token accounts                         (rents back to the relayer)
```

`T` ends with nothing. The output note's amount is learned from the DEPOSIT record.

**Relayer safety.** The browser builds the exact message with the SDK and signs it
with `T`; the relayer rebuilds the same message from the request parameters, attaches
`T`'s signature and its own, and simulates with signature checks before sending. It
never signs a message it did not build, and a failing transaction is never sent.

## 5. Admin, caps and trust

- Pools are created and capped only by the shield's **upgrade authority** (read from
  ProgramData), so moving that authority to a multisig moves admin too. Both programs
  stay upgradeable until audited.
- **Trusted setup.** Phase 1: Perpetual Powers of Tau (PSE) `ppot_0080_15.ptau`. Phase 2:
  one discarded random contribution plus a public beacon (a finalized Solana mainnet
  blockhash, `circuits/BEACON.txt`). More contributions can be added before raising caps.
- **Relayer** (`app/server/relay.ts`, a Vercel Function). It sees request IPs and the
  receiving wallet of withdrawals, never which deposit is spent.

## 6. Roadmap

- **Phase A (live)** — private account (SOL, USDC) + privacy hook; relayer; launch caps.
  The hook is permissionless: anyone can launch a private SOL/USDC Socket market with it
  (see `docs/HOOK.md`); the launch pool is `Gc6zJAtk…372fq`.
- **Phase A+** — private accounts for any SPL token (private markets for any pair), an
  in-app "launch a private market" flow and pool list, Socket allowlist for routing.
- **Phase B** — hidden swap sizes (sealed batches inside the private account, view keys),
  external audit, multisig authority, higher caps, optional association sets (prove
  funds are clean without revealing which deposit is yours).

*History:* the first mainnet deployment (`4pu6…aLTD`) was a commit–reveal hook; it
hid authorship proofs but not who traded, and was closed and replaced by this design.

---

*DARKHOOK — launch a market, hook in privacy, let it evolve privately.*
