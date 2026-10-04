# DARKHOOK

**The programmable privacy hook layer for onchain markets.**

- **Launch a market. Hook in privacy. Let it evolve privately.**

DARKHOOK makes a [Socket](https://thesocket.app) market private by rule:

1. **Deposit** SOL or USDC into your *private account* — a zero-knowledge shielded
   pool where balances are secret notes.
2. **Swap privately.** Your browser proves "one of the notes is mine", a one-time
   wallet swaps on a DARKHOOK pool, and the whole output goes back in as a new note.
   A relayer pays the fees; your wallet never signs and never appears.
3. **Withdraw** any part, to any wallet. What you leave stays private.

The **privacy hook** is what makes it a rule, not an option: Socket calls it before
every swap on the pool, and it refuses any swap that is not funded from the private
account and settled back into it. Full specification: **[PROTOCOL.md](./PROTOCOL.md)**.

**Docs:** [How DARKHOOK works](./docs/HOW-IT-WORKS.md) (no code) ·
[The hook, and how to launch your own private market](./docs/HOOK.md#3-launch-your-own-private-market) ·
[Protocol specification](./PROTOCOL.md)

---

## Repository layout

```
darkhook/
├── PROTOCOL.md                       protocol specification + threat model
├── crates/socket-hook-interface/     vendored mirror of Socket's hook ABI
├── programs/
│   ├── gate-hook/                    the privacy hook (stateless, ~40 KB)
│   │   └── src/lib.rs                  check_private: vault in → swap → vault out
│   └── shield/                       the private account
│       └── src/
│           ├── lib.rs                  InitPool / Deposit / DepositAll / Withdraw / SetCap
│           ├── merkle.rs               incremental Poseidon tree + root history
│           ├── groth16.rs, field.rs    verifier on the alt_bn128 syscalls
│           └── vk.rs, zeros.rs         generated from the circuit
├── circuits/                         spend.circom + trusted-setup scripts
├── sdk/                              @darkhook/sdk (TypeScript)
│   └── src/
│       ├── shield.ts                   notes, tree, spend witnesses, instructions, events
│       ├── private-swap.ts             the one-transaction private swap (v0 + lookup table)
│       ├── socket.ts                   Socket builders (pool, position, liquidity, swap)
│       ├── codec.ts                    HookCall / HookReply codecs
│       └── constants.ts                program ids, masks, mints
├── tests/integration/                e2e vs the mainnet Socket binary + setup scripts
└── app/                              website (one Vercel project)
    ├── index.html                      landing page        → /
    ├── trade/index.html + src/         private account app → /trade/
    ├── public/zk/                      circuit + proving key for in-browser proofs
    ├── api/rpc.ts                      RPC proxy (edge)    → /api/rpc
    ├── api/relay.js                    relayer (bundled from server/relay.ts) → /api/relay
    └── server/relay.ts                 relayer source
```

## Build & test

```powershell
cargo test --workspace     # gate hook (6) · shield (17, incl. a real Groth16 proof) · interface (5)
cd sdk; npm install; npm test   # 20 tests, pinned to circuits/vectors.json
```

End to end on a **local, throwaway** validator (Linux/WSL with the Agave toolchain and
Node ≥ 20) — Socket's program is cloned from mainnet read-only, the shield and the hook
are loaded at their SDK ids:

```bash
cargo build-sbf --manifest-path programs/shield/Cargo.toml --sbf-out-dir target/deploy
cargo build-sbf --manifest-path programs/gate-hook/Cargo.toml --sbf-out-dir target/deploy
bash tests/integration/run-private.sh target/deploy/shield.so target/deploy/gate_hook.so
bash app/test/run-private.sh target/deploy/shield.so target/deploy/gate_hook.so   # through the relayer; KEEP=1 serves the app
```

`run-private.sh` checks: a plain swap is refused (`NotFundedFromVault`), a vault-funded
swap that keeps its output is refused (`NotSettledToVault`), private swaps both ways
(user's wallet absent, throwaway wallet emptied), partial withdrawals with change,
double spends, inflated amounts, fee raising, payout redirection and forged change
notes. `app/test/run-private.sh` runs the mainnet setup script (dry run, real run,
idempotent re-run) and drives the real relayer handler like the browser does.

## Mainnet

| | |
|---|---|
| Private account (`shield`) | `35kTmDNFypu19FfRvJ2sWTwFySyBXvE6LC9Usz9uUwtF` — 93 256 bytes |
| Privacy hook (`gate`) | `GNRa1VQta9gMpBUfWHpe3nmSaScyDvNGu5KdprsXsr3p` — 40 136 bytes, built against the shield id above |
| Upgrade authority (both) | `3pSjvyKtMfGqyceyrS6gxda2WH9JtxKW6SD8JQFievqQ` |
| Private SOL / USDC pools | `6UoPKozWtZZSQxXDZVcUD75RhLxCZoNLo1RAHoeJHhSy` / `CigJ7XHyKYcBZmh5uAzuL2b4Mz8k8rmgP2XzCDS5ENsb` (caps 50 SOL / 5 000 USDC) |
| Gated Socket pool wSOL/USDC | `Gc6zJAtkg36uCbk3SAvtE8NKzR9yjqeocETGe1G372fq` (mask 1, extra = Instructions sysvar) |
| Lookup table | `GU6xCKFSQA5bCDbkmLYqasfPJXogAg5DmxkmX5XrV9NA` |
| Relayer | `CLz6dudyf9x9zKRgzaogC3ZVxEjSFaD2Z5T63No2UmU6` |
| Socket program | `24Qa8yMGZMuz3fqGAFPK36cu47d94o9iKv8GAKfbo54Q` |

Binary hashes: shield `b5a6088abdbbdfe6de828784fee5be694e088b4820de1a6371eb03201ed26ed3`, gate
`539e6e8195490c705c1623d279aea33fac345168a4eb4f89523c195127d53de4` (on-chain bytes verified
identical). The first commit–reveal hook `4pu6…aLTD` was closed.

## Deploying (Linux/WSL, toolchain installed)

Program keypairs fix the ids. The hook must be built against the shield's id:

```bash
solana-keygen pubkey keys/shield-program.json          # → SHIELD_ID
cargo build-sbf --manifest-path programs/shield/Cargo.toml --sbf-out-dir target/deploy
SHIELD_PROGRAM_ID=<SHIELD_ID> cargo build-sbf --manifest-path programs/gate-hook/Cargo.toml --sbf-out-dir target/deploy
solana program deploy target/deploy/shield.so    --program-id keys/shield-program.json -u mainnet-beta
solana program deploy target/deploy/gate_hook.so --program-id keys/gate-program.json   -u mainnet-beta
# then (dry run first, SEND=1 to execute):
WALLET=~/.config/solana/id.json RELAYER=<relayer pubkey> PRICE=<USDC per SOL> \
  LIQ_SOL=<sol> LIQ_USDC=<usdc> node tests/integration/private-setup.mjs
```

If the ids differ from the SDK defaults, set `SHIELD_PROGRAM` / `GATE_PROGRAM` for the
setup script and `VITE_SHIELD_PROGRAM_ID` / `VITE_GATE_PROGRAM_ID` for the app. The
setup script prints the remaining env (`VITE_PRIVATE_POOL`, `VITE_PRIVATE_ALT`, …).
Rent: shield ~0.47 SOL (93 KB), hook ~0.20 SOL (40 KB); each deploy refunds its buffer.
Verify a deployment byte-for-byte with `tests/integration/verify-deploy.sh <id> <so>`.

**On-chain naming.** The binaries carry no project name (`shield`, `gate`); check with
`strings target/deploy/*.so | grep -i dark` — it must print nothing.

## Security status

- **Unaudited; it custodies funds.** Launch caps (deposits minus withdrawals, set by the
  upgrade authority, never blocking withdrawals), upgradeable programs, internal review
  only. Phase-2 trusted setup: one contribution plus a public beacon — add more before
  raising caps (circuits/README.md).
- The hook moves no funds and holds no state; its only power is refusing a swap.
- Privacy limits (amount matching, small anonymity sets, public swap sizes): PROTOCOL.md §1.

## Background

Socket's insight: *a new AMM idea ships as a hook, not a new venue*. DARKHOOK takes the
same position for privacy: the privacy of a market is a hook with rules, plugged into
whatever pool you launch.

> **ZEROHOOK** — the programmable privacy layer for onchain markets.
> *Hook in privacy. Program the market.*

## License

MIT OR Apache-2.0.
