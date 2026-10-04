# How DARKHOOK works

DARKHOOK makes a market on Solana private **by rule**: on a DARKHOOK pool, nobody can
swap from a normal wallet. Every swap is paid out of a shared *private account* and
its result goes straight back into it, so the chain never shows whose swap it was.

This page explains the idea without code. The precise specification is in
[PROTOCOL.md](../PROTOCOL.md); the hook itself is explained in [HOOK.md](./HOOK.md).

---

## The problem

On a normal DEX every swap is signed by a wallet. Anyone can open an explorer and see
which wallet bought what, when, and how much — and follow that wallet everywhere else.
Using a "fresh" wallet doesn't help for long: the moment you fund it from your main
wallet, or send the profits back, the two are linked forever.

## The three pieces

```
           your wallet                                any wallet
               │ deposit                                    ▲ withdraw
               ▼                                            │
   ┌───────────────────────────────────────────────────────────────┐
   │                PRIVATE ACCOUNT  (zero-knowledge)               │
   │     balances are secret notes · nobody sees whose they are     │
   └───────────────┬───────────────────────────────▲───────────────┘
                   │ 1. pay the swap                │ 3. take the result back
                   ▼                                │
   ┌───────────────────────────────────────────────────────────────┐
   │          SOCKET POOL  SOL/USDC   (the market, public)          │
   │   ─── before every swap Socket asks ───►  DARKHOOK HOOK        │
   │                                          "did the money come   │
   │                                           from the private     │
   │                                           account, and does    │
   │                                           the result go back?" │
   │                                           no → swap refused     │
   └───────────────────────────────────────────────────────────────┘
```

1. **The private account** — a shielded pool (program `35kTmDNF…uUwtF`). When you deposit,
   your browser creates a secret *note* ("I own X SOL") and the account stores only a
   fingerprint of it. Nobody — not the website, not the relayer — can tell which note is
   yours. To spend a note you show a **zero-knowledge proof**: "I own one of the notes in
   this account", without saying which one.

2. **The market** — a normal [Socket](https://thesocket.app) pool trading SOL against USDC,
   with liquidity providers who earn fees. Socket is an AMM where each pool can plug in
   one *hook* program that Socket calls at fixed moments.

3. **The DARKHOOK hook** — a small program (`GNRa1VQt…sXsr3p`) plugged into the pool.
   Socket calls it **before every swap**. It reads the whole transaction and refuses the
   swap unless the money comes out of the private account and the result goes back in.
   Because a pool's hook is fixed when the pool is created, this rule can never be
   turned off for that pool.

Two helpers make it smooth:

- **A one-time wallet**, created in your browser for a single swap. It holds funds only
  inside that one transaction and ends with zero.
- **A relayer**, a server that pays the network fees and submits the transaction, so
  your wallet signs nothing. It takes a small fee from the note being spent. It cannot
  change where the money goes (the proof locks that) and it never learns which note is
  yours.

## A private swap, step by step

All of this happens in **one** Solana transaction, so it either fully happens or not at all:

1. **Prove.** Your browser proves you own a note worth, say, 2 SOL, and that you want to
   spend 1 SOL of it. The other 1 SOL (minus the fee) becomes a new hidden *change* note.
2. **Withdraw to a one-time wallet.** The private account pays 1 SOL to the one-time wallet.
3. **Swap.** The one-time wallet swaps 1 SOL for USDC on the Socket pool.
   Socket first asks the hook; the hook sees step 2 before and step 4 after, and says yes.
4. **Deposit back.** All the USDC goes back into the private account as a new note for you.
5. **Clean up.** Temporary accounts are closed; the one-time wallet ends empty.

Your browser then knows two new notes (1 SOL change, ~USDC output). Nobody else does.

## What people can and cannot see

| Visible to everyone | Hidden |
|---|---|
| That a wallet deposited into the private account, and how much | Which later swap or withdrawal that deposit paid for |
| That a swap of 1 SOL → USDC happened on the pool | **Who** made it — your wallet is not in the transaction |
| That someone withdrew an amount to some wallet | Which deposit it came from |
| The pool's prices and liquidity (it is a public market) | What is left in your account |

**Privacy grows with use.** If you are the only user, or you deposit 2.3471 SOL and
withdraw exactly 2.3471 SOL a minute later, an observer can guess. Use round amounts,
leave something inside, and let some time pass between steps.

## Your notes are your keys

Your balance lives in your browser as secret notes. Whoever has them can spend the
funds; if you lose them, the funds are gone. The app lets you **download a backup**
after every operation and **restore** it later or on another device.

## Safety status

- The programs are unaudited. The private account launches with **deposit caps**
  (50 SOL, 5 000 USDC) that limit how much can be at risk; the caps never block
  withdrawals.
- Both programs are upgradeable by the team until an external audit.
- The hook holds no money and has no switches: its only power is refusing a swap.

## Who can launch a private market?

Anyone. The hook is permissionless: any wallet can create a new Socket pool with the
DARKHOOK hook plugged in, add liquidity and earn its fees. Today such pools must trade
SOL against USDC — the assets the private account holds; private accounts for other
tokens are next on the roadmap. See [HOOK.md](./HOOK.md#3-launch-your-own-private-market).
