# ORBIT Ajo Protocol

**ORBIT** is a rotating savings & credit association (ROSCA — locally known
as **"Ajo"**) built on [Stellar Soroban](https://soroban.stellar.org/), with
staked collateral and member-voted default slashing to make trustless group
savings viable on-chain.

A group of members ("an orbit") each contribute a fixed amount every round;
each round, the full pot is paid out to one member, until everyone has been
paid once. ORBIT adds two things a plain ROSCA doesn't have: **collateral
staked upfront** (so a member can't walk away after receiving their payout
without also losing something), and **on-chain dispute resolution** where
the group itself votes to slash a defaulter's stake.

## Repositories

| Repo | What it is |
|---|---|
| [**orbit-contracts**](https://github.com/ORBIT-Ajo-Protocol/orbit-contracts) | Soroban smart contracts — the protocol logic itself |
| [**orbit-backend**](https://github.com/ORBIT-Ajo-Protocol/orbit-backend) | Indexer + REST/WebSocket API that mirrors on-chain state, and a mock SEP-24 anchor for the NGN↔USDC demo flow |
| [**orbit-frontend**](https://github.com/ORBIT-Ajo-Protocol/orbit-frontend) | React/Vite demo UI — member app + admin portal |

## How the pieces fit together

```
                 ┌────────────────────┐
  create_orbit → │   orbit-factory     │  deploys one orbit-contract
                 │  (one per protocol) │  instance per group, tracks
                 └─────────┬──────────┘  all deployed addresses
                            │ deploy_v2
                            ▼
                 ┌────────────────────┐
                 │   orbit-contract    │  one instance per savings
                 │  (one per group)    │  group: membership, stake,
                 └─────────┬──────────┘  contributions, payouts, disputes
                            │ getEvents (poll)
                            ▼
                 ┌────────────────────┐
                 │   orbit-backend     │  indexes events → Postgres,
                 │ (indexer + API +ws) │  serves REST reads, builds
                 └─────────┬──────────┘  unsigned tx XDR (non-custodial)
                            │ (not yet wired)
                            ▼
                 ┌────────────────────┐
                 │   orbit-frontend    │  member app + admin portal
                 │  (fully simulated)  │  — currently faked client-side
                 └────────────────────┘
```

**Current status:** contracts and backend are deployed and tested
end-to-end against Stellar testnet. The frontend is a complete UI/UX
simulation that has not yet been wired to either — see each repo's README
for exact status and testnet contract addresses.

## Protocol flow (on-chain)

1. **`orbit-factory.create_orbit`** — anyone can deploy a new group,
   configuring token, contribution amount, frequency, payout order
   (`Fixed` / `Random` / `Auction`), stake percentage, round count, and
   grace period.
2. **`add_member`** (admin) — seats members at rotation slots while the
   group is `Pending`.
3. **`lock_stake`** (member) — each member posts their collateral
   (`contribution_amount × stake_bps`) upfront, before the group can go
   live. This is what makes slashing meaningful even against a member who
   defaults on round 1.
4. **`activate`** (admin) — once every slot is filled and staked, the group
   goes `Active` and round 1 starts.
5. **`contribute`** (member, every round) — once every active member has
   contributed, the round auto-settles in the same transaction: the pot is
   paid to that round's recipient (fixed order, weighted-random, or lowest
   auction bid, depending on config).
6. **`propose_dispute` → `vote_slash` / `finalize_dispute`** — if a member
   misses the grace period, any other member can raise a dispute; the group
   votes, and an approved dispute slashes the defaulter's stake into the
   pot and marks the round settled so the cycle isn't stuck.

Full method-level detail, testnet contract addresses, and build/deploy
instructions live in [orbit-contracts](https://github.com/ORBIT-Ajo-Protocol/orbit-contracts).

## Getting started

Each repo is independently runnable — see its README:

- [orbit-contracts](https://github.com/ORBIT-Ajo-Protocol/orbit-contracts#build-order) — Rust + `stellar` CLI
- [orbit-backend](https://github.com/ORBIT-Ajo-Protocol/orbit-backend#running-locally) — Docker Postgres + Node
- [orbit-frontend](https://github.com/ORBIT-Ajo-Protocol/orbit-frontend#run-locally) — Node, no env vars required
