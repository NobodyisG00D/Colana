# SIMD Stack

Colana is deployed on top of three Solana governance proposals that together enable programmable validator revenue:

| Proposal | Name | Role |
|:---|:---|:---|
| SIMD-0185 | Vote Account v4 | Split inflation vs block revenue commission |
| SIMD-0123 | Block Revenue Distribution | Protocol-enforced delegator fee sharing |
| SIMD-0232 | Custom Commission Collector | Route commission to program-controlled PDAs |

## Vote account fields

- `inflation_rewards_commission_bps`
- `block_revenue_commission_bps`
- `inflation_rewards_collector`
- `block_revenue_collector`

## Collector constraints (SIMD-0232)

Collectors must be System Program owned, rent-exempt, and not reserved — or commission is **burned**.
