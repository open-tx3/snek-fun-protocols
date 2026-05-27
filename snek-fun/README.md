# snek.fun

[snek.fun](https://snek.fun/) is a fair-launch token launchpad on Cardano. Anyone can launch a new token at the same fixed starting valuation; the token trades on its own bonding-curve pool from the moment it exists, prices rise with demand, and once the token reaches a 69 000 ADA market cap it **graduates** — liquidity migrates to [Splash DEX](https://splash.trade/) and the LP position is burned, locking the liquidity forever.

The launchpad is designed around the same playbook popularised by Solana's pump.fun: zero pre-sale, no team allocation, no liquidity bootstrapping events. Token creators pay a small launch fee, get a "dev buy" allocation, and the bonding curve handles price discovery. Traders use snek.fun's UI to buy or sell into the curve until graduation. See [docs.snek.fun](https://docs.snek.fun/getting-started/introduction) for the protocol overview.

This tx3 covers the **user-facing flow** of the v1 protocol: placing buy / sell orders against a token's bonding curve, cancelling orders, and launching new tokens. Pool fills (the actual bonding-curve trades) are executed by snek.fun's permitted-executor batcher and are out of scope.

## Overview

Each token launched on snek.fun has its own bonding-curve pool. Trades follow a two-step model:

1. **User submits an order** — sends ADA (buy) or tokens (sell) to a per-user order validator address with an inline datum describing the desired swap.
2. **Permitted executor processes the order** — the snek.fun batcher consumes the order UTxO together with the bonding-curve pool UTxO, fills the trade, and returns funds to the user.

Scope of this tx3:

- Implemented: user-side `place_buy_order`, `place_sell_order`, `cancel_order`, plus the admin-ish `launch_token` (mint + seed + metadata + fee in one tx).
- Not implemented: pool spends. Filling an order requires the permitted-executor signature plus dynamic input/output indices against the pool UTxO — both blocked by current tx3 limitations.

## Transactions

| Transaction | Description |
|---|---|
| `place_buy_order` | Deposit ADA at the order validator; the batcher will return tokens |
| `place_sell_order` | Deposit tokens at the order validator; the batcher will return ADA |
| `cancel_order` | Spend a user's own order UTxO back to their wallet (`Cancel` redeemer) |
| `launch_token` | Mint a new token (1B supply), seed its bonding-curve pool, deposit metadata, and pay the launch fee |

## Important considerations

- **Order submissions are script-free.** `place_buy_order` and `place_sell_order` are plain payments to the order validator address with an inline datum — no validator runs. The script only fires when the batcher fills or the user cancels.
- **Per-user order address.** `OrderScript` is `(payment = order validator script, stake = user's own stake key)`. Because the stake side changes per user it cannot live in env — callers must pass the concrete bech32 per call as the `orderscript` party.
- **Per-launch token policy.** `launch_token` requires a parameterised minting policy (new policy id for every token). The caller must apply the seed outref to the on-disk template off-chain and pass both the resulting `token_policy` (script hash) and `token_script` (CBOR) per call.
- **Flat address structure.** `OrderDatum.owner_addr` is modelled as a chain of single-field structs (`CardanoAddress → OrderPayment + OrderStakeJust → OrderStakeCred → OrderPayment`) to reproduce Cardano's nested `Constr(0, ...)` address layout while sidestepping the tx3 resolver's current issues with deeply-nested enum variants.
- **Pre-summed ADA amounts.** tx3 has no `*` or `/`, so every aggregate (`total_escrow_ada`, `pool_seed_ada`, `creator_min_ada`, etc.) must be pre-computed by the caller. The launch tx alone takes 6 separate pre-summed figures.
- **Direction marker fixed at 1.** `OrderAmount.direction` is hardcoded to `1` (exact-input variant). The `direction=0` "buy-with-output" mode is not implemented.
- **Pool spends not implemented.** Buy/sell fills against the bonding curve are done by the snek.fun batcher and require the permitted-executor signature + dynamic input/output indices — they cannot be modelled in tx3 today.

## Caller preparation

### `place_buy_order`

| Parameter | Source |
|---|---|
| `orderscript` party | Bech32 of `(order validator script + user's stake key)`. Must be assembled per user. |
| `user_pkh`, `user_stake_key` | Raw 28-byte hashes from the user's wallet, also embedded in the datum's nested owner address. |
| `token_policy`, `token_name` | The token information the user is buying (queried from snek.fun for that pool). |
| `ada_input` | Lovelace the batcher may spend on tokens (caller chooses based on slippage tolerance). |
| `total_escrow_ada` | `ada_input + executor_fee + min_order_out_ada`. Pre-summed. |
| `deadline_ms` | Unix millis after which the batcher rejects the order. |
| `empty_bytes` | Placeholder `""` for the ADA asset id (`policy = ""`, `name = ""`). |

### `place_sell_order`

Same as `place_buy_order` plus:

| Parameter | Source |
|---|---|
| `token_amount` | Token quantity being sold. |

The escrow ADA is fixed (`sell_escrow_ada` from env, observed = 2 600 000 lovelace) so callers don't pass it.

### `cancel_order`

| Parameter | Source |
|---|---|
| `order_utxo: UtxoRef` | The order UTxO to reclaim. The user must also have a separate pure-ADA UTxO available for fees and collateral. |

### `launch_token`

The most parameter-heavy transaction. The caller precomputes everything off-chain.

| Parameter | Source |
|---|---|
| `seed_utxo`, `seed_tx`, `seed_idx` | A spendable UTxO from the creator's wallet — used to parameterise the token policy and as the input to the pool NFT mint redeemer. |
| `token_policy`, `token_script` | Apply the on-disk token mint template (`investigacion/scripts/token_mint.v3.template.cbor.hex`) to the seed outref. The resulting blake2b-224 hash is `token_policy`; the applied CBOR is `token_script`. |
| `pool_nft_name` | 32-byte hash the pool NFT policy expects, derived from `(seed_outref_tx, seed_outref_idx)`. |
| `metadata_nft_name`, `ticker`, `logo_cid`, `description`, `launch_type`, socials, `metadata_version` | Token metadata — passed as raw bytes (hex-encoded UTF-8 for text fields). |
| `creator_pkh`, `creator_stake_key`, `pool_witness_pkh` | Wallet identity values, recorded in both the pool datum and metadata datum. |
| `ada_cap_thresh_for_pool` | Per-pool graduation threshold (close to `18_188_400_000` ± per-launch jitter). |
| `launch_fee_ada` | Fee paid to the snek.fun collector (observed = 1 825 000). |
| `metadata_min_ada`, `creator_min_ada`, `pool_seed_ada` | Min-ADA values for the metadata, creator, and pool outputs. |
| `initial_buy_tokens`, `curve_tokens_remaining` | Split of the 1 000 000 000 supply between the creator's "dev buy" and the bonding-curve seed. Must sum to `token_emission`. |

Token policy application snippet (Python):

```python
import hashlib
tpl = bytes.fromhex(open('investigacion/scripts/token_mint.v3.template.cbor.hex').read())
seed_tx  = bytes.fromhex('<your seed tx hash>')   # 32 bytes
seed_idx = 0                                       # 0..23
applied  = tpl[:475] + seed_tx + bytes([seed_idx]) + tpl[475+33:]
policy   = hashlib.blake2b(b'\x03' + applied, digest_size=28).hexdigest()
# feed `applied.hex()` as token_script and `policy` as token_policy
```

## References

- **Smart contracts:** PlutusV2 order validator (reference script at `e2ed9e953ebf98ca701fc93588d73cb9769f87b9d13712474f566a0743963e8b#0`) + PlutusV2 bonding curve & pool NFT policy + PlutusV3 per-launch parameterised token mint. Source closed; on-chain shapes reverse-engineered.
- **Homepage / app:** [snek.fun](https://snek.fun/)
- **Docs:** [docs.snek.fun](https://docs.snek.fun/getting-started/introduction)
- **Migration target on graduation:** [Splash DEX](https://splash.trade/)
- **Research notes:** [`investigacion/snek-fun-research.md`](./investigacion/snek-fun-research.md)
