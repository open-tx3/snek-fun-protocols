# snek.fun protocol — on-chain research

A Cardano memecoin launchpad by the Snek and Splash Protocol teams.
- Docs: https://docs.snek.fun/
- App: https://www.snek.fun/
- Builder HTTP API: https://builder.snek.fun
- Analytics HTTP API: https://analytics.snek.fun

All tokens launched via snek.fun follow the same on-chain shape:
1. A bonding curve pool UTxO holds an instance-specific pool NFT + remaining
   token supply + ADA reserves. Prices derive from constants `aNum/bNum`.
2. A separate order validator holds batched buy / sell orders. Off-chain the
   snek.fun "Permitted Executor" (batcher) matches orders against pools.
3. At market cap ≥ `adaCapThreshold` the pool graduates to Splash DEX and
   LP tokens are burned.

## Live protocol parameters (GET https://builder.snek.fun/parameters)

```json
{
  "tokenEmission":   "1000000000",             // 1,000,000,000  (1 B supply per launch)
  "aNum":            "122525779519",
  "aDenom":          "1000000000000000000000000000",  // 1e27
  "bNum":            "2545182",
  "bDenom":          "1000000",                // 1e6
  "adaThreshold":    "18188400000",            // ≈ 18 188 ADA, graduation threshold (lovelace)
  "fixedFee":         "500000",                // 0.5 ADA — per trade flat fee
  "minTradeAda":     "1000000",                // 1 ADA — minimum trade
  "percentFee":       1,                        // 1% trade fee
  "deployScript":    "905ab869961b094f1b8197278cfe15b45cbe49fa8f32c6b014f85a2d"
}
```

## Stable on-chain identifiers (mainnet v1)

| Role | Value |
|------|-------|
| Bonding curve validator (script hash) | `905ab869961b094f1b8197278cfe15b45cbe49fa8f32c6b014f85a2d` |
| Bonding curve address (script+script) | `addr1xxg94wrfjcdsjncmsxtj0r87zk69e0jfl28n934sznu95tdj764lvrxdayh2ux30fl0ktuh27csgmpevdu89jlxppvrs2993lw` |
| Bonding curve script ref UTxO | `c4a540ac2e06...#3` (plutusV2, ~2000 bytes) |
| Order validator (script hash) | `d9143ac63473b17a215d1b7484dfb6ac6b4a0005beb0e26a6ca02c96` |
| Order validator ref UTxO | `e2ed9e953ebf...#0` (plutusV2, ~1076 bytes) |
| Withdraw/staking validator (used by pool spends) | `a5643b4a22a192d7691d05baf4a9bbb8acdbb5daa60be1f333e128f1` |
| Withdraw validator ref UTxO | `e2ed9e953ebf...#1` (plutusV2, ~1351 bytes) |
| Pool NFT minting policy (stable across pools) | `63f947b8d9535bc4e4ce6919e3dc056547e8d30ada12f29aa5f826b8` |
| Metadata validator address | `addr1q8lsjfvtpnu9kv5zhwgsdcw03tlkuwcvjqmzm35arx9xl6k6s0uhca7762y56q2j9en5ttn69p7a048cw3mz62fuj3mqauv0x3` (payment cred `ff09258b0cf85b3282bb9106e1cf8aff6e3b0c90362dc69d198a6fea`) |
| Launch fee collector | `addr1qy32agk6zhjffcqhvu296j9a594k6smlr3zfsgaqgsvnmtefn2pw7433r2ss4hcycqrj6jrsaw056hlnz5fjgdyyrd6qsw8ngt` (cred `22aea2da15e494e01767145d48bda16b6d437f1c449823a044193daf`) |
| Permitted executor PKH (batcher, signs order fills) | `e865941988edcca559268b57b7ee939974fd42fd26c7e1acd7a50678` |
| Factory witness PKH (appears in every pool datum) | `8807fbe6e36b1c35ad6f36f0993e2fc67ab6f2db06041cfa3a53c04a` |

### Order validator address per user

Orders sit at an address with the order validator as *payment* credential and
the **user's own stake key** as the stake credential. Example live bech32s:

- `addr1z8v3gwkxx3emz73pt5dhfpxlk6kxkjsqqkltpcn2djsze94s2hu5cuh44qs` (user A)
- `addr1z8v3gwkxx3emz73pt5dhfpxlk6kxkjsqqkltpcn2djsze9sn0uvsrqde7w8` (user B)

Both share the 28-byte script payment `d9143ac6...`. The stake segment
changes per user. In tx3 we accept this as a per-call parameter
(`order_target_address`) since it cannot be encoded as a static party.

## Datum shapes

### PoolDatum (locked at bonding curve address, 9 fields, Constr 0)

```text
0: Constr 0 { pool_nft_policy: Bytes(28), pool_nft_name: Bytes(32) }  // pool beacon NFT
1: Constr 0 { bytes "", bytes "" }                                    // quote asset = ADA
2: Constr 0 { token_policy: Bytes(28), token_name: Bytes }            // base asset
3: Int  aNum         (== 122_525_779_519 on v1)
4: Int  bNum         (== 2_545_182 on v1)
5: Bytes permittedExecutorPKH (== e865941988edcca559268b57b7ee939974fd42fd26c7e1acd7a50678)
6: Int  adaCapThreshold   (≈ 18_188_400_000 — slight per-pool jitter)
7: Bytes factoryWitnessPKH (== 8807fbe6e36b1c35ad6f36f0993e2fc67ab6f2db06041cfa3a53c04a)
8: Bytes adminWitnessPKH   (appears stable inside a deployment; varies per pool type)
```

### OrderDatum (locked at order validator address, 10 fields, Constr 0)

```text
0: Bytes  "01"                                             // version/marker flag
1: Constr 0 { owner_address }                              // wrapped Cardano Address
       Constr 0 {
         payment: Constr 0 { bytes pkh },                 // PubKey credential
         stake: Constr 0 {                                // Option::Some
           Constr 0 {                                     // Inline
             Constr 0 { bytes pkh }                       // PubKey credential
           }
         }
       }
2: Constr 0 { policy: Bytes, name: Bytes }                 // input asset  (what user pays)
3: Constr 0 { policy: Bytes, name: Bytes }                 // output asset (what user wants)
4: Constr 0 { direction: Int, amount: Int }                // 1 = exact-input, 0 = exact-output
5: Int  min_output_ada        (== 1_100_000 — min ADA that goes back to user)
6: Int  executor_fee          (== 1_500_000 — tip for the batcher)
7: Bytes permittedExecutorPKH (== e865941988edcca559268b57b7ee939974fd42fd26c7e1acd7a50678)
8: Int  deadline              (unix millis)
9: Bytes owner_pkh            (duplicate of field 1.payment pkh, for fast indexing)
```

- **BUY**  → field 2 = ADA, field 3 = token, UTxO value = amount ADA + 1.5 ADA fee + minAda.
- **SELL** → field 2 = token, field 3 = ADA, UTxO value = amount tokens + 2.6 ADA or so.
- `BUY_WITH_OUTPUT` uses direction = 0 (pay variable ADA for fixed token output).

### MetadataDatum (locked at metadata validator during launch)

```text
0: Constr 0 { token_policy, token_name }                   // asset id
1: Bytes ticker                                            // e.g. "Poppy"
2: Bytes ipfs_cid                                          // logo
3: Bytes description
4: Int  launchType                                         // 0 Meme, 1 Hyped
5: Constr 0 { poolAuthor_pkh, poolAuthor_skh }
6: Constr 0 { twitter_url, discord_url, telegram_url, website_url }  // bytes each, "" if none
7: Int  version / extra flag (observed = 1)
```

## Redeemers

### Pool spend (at bonding curve address)
```
Constr 0 [ Int, Int, Constr 0 [] ]
```
Observed values:
- `[0, 0, ()]` for BUY fills (tx 4f8635de)
- `[1, 1, ()]` for SELL fills (tx 693dae40, 9d476fc3)

These appear to be (order_input_idx, own_output_idx) — used so the validator
can locate the matched order and the updated pool output. Per-call data; the
batcher populates them.

### Order spend (at order validator address)
```
Constr 1 []   // "Execute"  — used by the batcher to consume the order
Constr 0 []   // "Cancel"   — used by the owner to reclaim the escrow
```

### Withdraw validator (pool-spend helper)
```
List []   // empty/unit — present as a 0-ADA reward withdrawal alongside every pool spend
```

### Minting (launch tx)
- Pool NFT mint redeemer: `Constr 0 [ Constr 0 [ bytes seed_tx_hash ], Int seed_idx ]`
- Token (and metadata NFT, shared policy) mint redeemer: `Constr 0 []`

## Reference transactions captured during research

| tx | description | token |
|----|-------------|-------|
| `87edffc1405348824bbe75adeb9df21d19e460fd13ed47da1072018bc0665125` | Legacy launch (sep 2024) | SNIGGA |
| `7e7161f3d5906ff39c83b71be97bce31324d611208287bffd21947e323ffc4d9` | Current v1 launch | Poppy (Ranch Mascot) |
| `5322698e22bf9db9f17c424c79a42861364b3ddf062eb4cb3bc303869a6250d4` | Place BUY order | Poppy |
| `693dae40b62bc729e6433d905c06cc4c2944f6c51ece40e7b708ee365dc1fd35` | BUY fill  (batcher) | Poppy |
| `af955807d66ca72976ae1c6846c7b4a91d4284fd72e541432f2f8a34249079d6` | Place SELL order | Poppy |
| `4f8635dea99905bbfbce0325969891fbde7f8fbe577cbbae7d8e01f147355b45` | SELL fill (batcher) | Poppy |

## Scope for the tx3 implementation

A complete snek.fun implementation touches four user-facing transactions:

1. **launch** — mint pool NFT + token + metadata NFT, seed pool with initial
   liquidity, record metadata. Complex: every launch has a *new* minting policy
   parameterised by the seed outref, so off-chain code must derive the policy id
   before the tx is built. Deferred: snek.fun exposes a builder at
   `POST https://builder.snek.fun/launch` that handles this.
2. **place_buy_order** — create an order UTxO at the order validator address
   (user's stake part), depositing ADA. No script execution, plain payment with
   an inline datum.
3. **place_sell_order** — same pattern but deposits tokens.
4. **cancel_order** — spend the user's own order UTxO using the order validator
   reference script with `Cancel` redeemer, refunds the escrow.

This research/implementation covers 2, 3, and 4 (the "order flow"). Launch is
documented above for reference but left to the builder API in this tx3 pass.

## Bonding curve math (for reference)

From the docs:
- Launch market cap starts at 2 550 ADA.
- Graduation at 69 000 ADA market cap (but the on-chain `adaCapThreshold` is
  measured against *ADA reserves in the pool*, not market cap — value
  ≈ 18 188 ADA).
- Formula not published; but `price = a * exp(b * sold)` fits the shape
  (`aNum / aDenom`, `bNum / bDenom`). Off-chain batcher computes the exact
  quote; the on-chain validator just checks the invariant.
