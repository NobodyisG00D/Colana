# Architecture

Colana is the on-chain distribution layer for **Solana version 2.0** — the architecture ChatGPT 5.6 produced when asked to rebuild Solana from first principles.

## Layers

1. **Solana Runtime** — SIMD-0123 / SIMD-0232 deposit commission into collector PDAs automatically
2. **Colana Program (`…BANK`)** — collector PDAs, reward vault, distribution logic
3. **Staker PDAs** — per-user, per-pool balance, lock expiry, reward debt
4. **User wallet** — claims SOL via `claim_rewards`

## Revenue path

```
Runtime → collector PDA → sync_rewards → reward index → claim_rewards → user
```

## Staking path

```
User → stake() → token vault PDA → staker PDA → claim_rewards() → SOL
```

## Program ID

`aZPKJ99yjfE6MjZ5isXNjm3ndanBdx5gX35nwmLBANK`
