<p align="center">
  <img src="./assets/logo.png" alt="Colana logo" width="120">
</p>

<h1 align="center">Colana</h1>

<p align="center">
  ChatGPT 5.6 rebuilt Solana in its own image — and called it <strong>Solana version 2.0</strong>.
</p>

<p align="center">
  <a href="https://solscan.io/account/aZPKJ99yjfE6MjZ5isXNjm3ndanBdx5gX35nwmLBANK">Mainnet Program</a> ·
  <a href="./docs/proposal.html">Full Proposal</a>
</p>

---

## What is Colana?

**Colana** started as a prompt: we asked ChatGPT 5.6 to rebuild Solana from scratch, in its own image — not as a fork, not as a whitepaper, but as a working on-chain system. What came back was a full redesign of how validator revenue, fee flow, and holder economics should work. The model called it **Solana version 2.0**.

We did not stop at the chat log. We took the architecture ChatGPT specified — custom commission collectors, protocol-enforced block revenue routing, program-controlled distribution PDAs, per-mint staking vaults — and deployed it on Solana mainnet using the real SIMD stack:

- **SIMD-0185** — Vote Account v4
- **SIMD-0123** — Block Revenue Distribution
- **SIMD-0232** — Custom Commission Collector Account

**Colana** is the token and thesis wrapped around that deployment: if an LLM rebuilds Solana in its own image, you get Solana 2.0 — programmable fee custody, on-chain yield, and zero trust in operator hot wallets.

---

## Why this matters

For most of Solana's history, validator income followed a rigid path:

- **Inflation commission** accumulated in the vote account
- **Block revenue** (base fees + priority fees) went to the validator identity hot wallet
- Any revenue sharing with holders happened **off-protocol** — manual transfers, Discord promises, dashboards you had to trust

ChatGPT 5.6 identified the same three problems when asked to redesign the chain from first principles:

1. **Trust, not code** — operators could promise fee sharing and never deliver
2. **No programmable collectors** — revenue could not route into PDAs for automated distribution
3. **No native meme yield** — pump.fun tokens had no standard way to earn from validator economics

The model's Solana 2.0 spec maps one-to-one onto the SIMD stack humans already voted into mainnet. ChatGPT independently converged on the same architecture. We shipped the collector program it specified.

**We built what ChatGPT designed. Colana is the token on top.**

---

## What we built

Colana combines three things that have never been wired together in production before:

1. **A validator vote account upgraded to v4**, with independent inflation and block revenue commission settings, each pointing at a custom collector PDA — not the identity hot wallet
2. **An on-chain Anchor program** (program ID ending in `BANK`) that receives protocol-level commission deposits and executes distribution rules without human intervention
3. **A memecoin staking layer** where holders stake pump.fun tokens (Token-2022) into program-controlled vaults and earn SOL yield from real validator revenue

| Dimension | Solana 1.0 | Colana / Solana 2.0 |
|:---|:---|:---|
| Architecture source | Human governance (SIMD votes) | ChatGPT 5.6 first-principles redesign |
| Commission custody | Identity hot wallet | On-chain program collector |
| Block revenue share | Off-protocol / trust-based | Protocol-enforced via SIMD-0123 |
| Custom distribution | Manual scripts | Contract instructions |
| Meme token staking | Not supported natively | Per-mint pools with vault PDAs |
| Revenue transparency | Opaque | Verifiable on-chain inflows |
| Admin key for payouts | Effectively yes | No — instruction-gated claims |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Solana Runtime (SIMD-0123 / SIMD-0232)         │
│  Deposits commission into collector PDAs        │
└──────────────────────┬──────────────────────────┘
                       │ lamports (automatic)
                       ▼
┌─────────────────────────────────────────────────┐
│  Colana Program (…BANK)                         │
│  Collector PDAs · Reward vault · Distribution   │
└──────────────────────┬──────────────────────────┘
                       │ claim_rewards
                       ▼
┌─────────────────────────────────────────────────┐
│  Staker PDAs (per user, per pool)               │
└──────────────────────┬──────────────────────────┘
                       │ SOL transfer
                       ▼
┌─────────────────────────────────────────────────┐
│  User wallet                                    │
└─────────────────────────────────────────────────┘
```

**Staking path:**

```
User wallet ──stake()──▶ Token vault PDA (per pool)
                              │
                              ▼
                     Staker state PDA (per user)
                              │
                              ▼
                     claim_rewards() ──▶ SOL to user
```

---

## On-chain proof

| | |
|---|---|
| **Program ID** | [`aZPKJ99yjfE6MjZ5isXNjm3ndanBdx5gX35nwmLBANK`](https://solscan.io/account/aZPKJ99yjfE6MjZ5isXNjm3ndanBdx5gX35nwmLBANK) |
| **Vanity suffix** | `BANK` — the treasury the runtime deposits commission into |
| **State accounts** | Thousands of on-chain accounts (pool + staker config, token vaults) |
| **Program balance** | ~946 SOL at time of writing (user activity + validator inflows) |

The program is deployed via the BPF Upgradeable Loader. Distribution rules are enforced at the instruction level — if an instruction does not pass the program's checks, the transfer does not happen.

---

## Instruction set

| Instruction | Signer | Effect |
|:---|:---|:---|
| `initialize_config` | Authority | One-time program setup |
| `initialize_pool` | Authority | Create a staking pool for a given mint |
| `stake` | User | Deposit meme tokens into vault; update staker PDA |
| `unstake` | User | Return tokens after lock period expires |
| `fund_rewards` | Anyone | Permissionless SOL top-up to reward vault |
| `claim_rewards` | User | Pro-rata SOL payout from reward index |
| `update_pool` | Authority | Modify pool parameters |
| `pause` / `unpause` | Authority | Emergency circuit breaker |

Commission deposits from the runtime require no instruction — lamports arrive automatically. An off-chain indexer or crank calls `sync_rewards` to update the global reward index when new commission lands in the collector PDAs.

---

## Account model

| Account | Seeds | Purpose |
|:---|:---|:---|
| Global config | `["config"]` | Program-wide parameters, pause flag |
| Pool state | `["pool", mint]` | Lock duration, total staked, reward index |
| User stake | `["stake", pool, user]` | Balance, deposit slot, reward debt |
| Token vault | `["vault", mint]` | Holds staked meme tokens |
| Reward vault | `["rewards", mint]` | Holds SOL rewards awaiting claim |
| Collector (inflation) | `["collector", "inflation"]` | Receives inflation commission from runtime |
| Collector (block) | `["collector", "block"]` | Receives block revenue commission from runtime |

---

## Validator configuration

Our vote account is upgraded to **Vote Account v4** per SIMD-0185:

| Field | Our configuration |
|:---|:---|
| `inflation_rewards_commission_bps` | Set independently from block revenue |
| `block_revenue_commission_bps` | Set independently from inflation |
| `inflation_rewards_collector` | Points at program collector PDA |
| `block_revenue_collector` | Points at program collector PDA |

Once configured:

- **Each epoch** — inflation commission deposits into `inflation_rewards_collector`
- **Each block** — block revenue commission deposits into `block_revenue_collector`
- **Under SIMD-0123** — non-commission block revenue distributes to delegated stake accounts automatically

---

## What's next

- **Permissionless pool creation** — any token deployer can `initialize_pool` without approval
- **Creator fee routing** — pump.fun creator fee streams as an additional reward source
- **Multi-validator support** — other v4 validators can point collectors at the Colana program
- **Publish the full ChatGPT 5.6 spec** — prompt, model output, and diff against Solana 1.0

---

## Security

- **Unaudited** — early-stage software; upgrade authority held by deployer key (multisig planned post-audit)
- **Collector PDA validity** — invalid collectors cause runtime to **burn** commission, not revert
- **Commission changes** — subject to SIMD-0249 epoch delay
- **No admin withdraw** — vault PDAs only release tokens via `unstake` when lock conditions are met

---

## Disclaimer

Colana is experimental memecoin infrastructure. Nothing here is investment, legal, or financial advice. Smart contract risk is real. Do your own research.

---

<p align="center">
  <img src="./assets/logo.png" alt="Colana" width="48">
  <br>
  <em>Solana 1.0 was built by humans. Solana 2.0 was designed by ChatGPT. Colana runs it on mainnet.</em>
</p>
