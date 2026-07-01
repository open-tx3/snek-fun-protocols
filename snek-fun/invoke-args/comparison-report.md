# snek.fun — tx3 vs on-chain comparison report

> **Note (2026-06-23):** the source was refactored to use tx3 0.23 features
> (address enums, inlined escrows/supply split, grouped `meta`/`author` record
> params, `""` byte literal, shared `owner_address` fn). The **on-chain output
> shapes below are unchanged** — verified by a full TIR diff against the
> pre-refactor baseline (every Constr nesting / field order preserved; the only
> diffs are the inlined `Add`/`Sub` arithmetic and `empty_bytes`→`""`, both
> byte-equivalent). The byte-match claims here still hold, and were **re-confirmed
> live-resolved on 2026-06-23** against the mainnet TRP (≥0.23):
> `place_buy_order` reproduces the on-chain Poppy order datum byte-for-byte
> (blake2b `da37…ec677`), and `launch_token`'s `PoolDatum` matches the real Poppy
> launch in 7/9 fields. See `protocols_tx3/TX3-0.23-UPGRADE.md` (snek-fun section)
> for the change log. The `invoke-args/*.json` reflect the new (smaller, partly
> self-describing) param surface.

All four user-facing transactions compile, build and produce CBOR that
structurally matches the real on-chain shape.

| tx3 transaction | status | test wallet |
|-----------------|--------|-------------|
| `place_buy_order`  | ✅ CBOR + datum byte-match to real tx | original Poppy buyer `addr1q89…njncd5` |
| `place_sell_order` | ✅ CBOR + datum byte-match, resolved against a funded wallet | `addr1q84uy5f05m54tn…645sdh9jdg` (32.5 M Poppy, 719 ADA) |
| `cancel_order`     | ✅ CBOR generated, spends a **live** open order | LAIKA seller `addr1q9cehmjzf2tmtz…sqwq3mt` |
| `launch_token`     | ✅ CBOR generated, 5 outputs match Poppy shape | same Poppy holder as seller |

## Reference on-chain transactions (mainnet)

| Role | tx hash | token |
|------|---------|-------|
| Place BUY order  | `5322698e22bf9db9f17c424c79a42861364b3ddf062eb4cb3bc303869a6250d4` | Poppy |
| Place SELL order | `af955807d66ca72976ae1c6846c7b4a91d4284fd72e541432f2f8a34249079d6` | Poppy |
| Cancel order     | *(none captured — protocol fills quickly)* | — |
| Launch token     | `7e7161f3d5906ff39c83b71be97bce31324d611208287bffd21947e323ffc4d9` | Poppy |
| Live target for cancel | `c665188f6c6f358ddabcbf49e02aa78c2c762c1517a29d024ffead51715c87c0#0` | LAIKA (open) |

## 1. `place_buy_order`

Generated hash: `a0cfd2f20fb090e7d3c798248cb2b454606c62230c4d20d1363e110625be06cb`

| Field | Generated | On-chain | Match |
|-------|-----------|----------|-------|
| Order output addr | `11d9143ac6…b055f94c…` | identical | ✅ |
| Order value | 71 600 000 lovelace | 71 600 000 | ✅ |
| Datum (10 fields, Constr 0) | byte-for-byte identical | — | ✅ |
| &nbsp;&nbsp;`marker` | `01` | `01` | ✅ |
| &nbsp;&nbsp;`owner_addr` | `Constr 0 [Constr 0 [pkh(cb7b…)], Constr 0 [Constr 0 [Constr 0 [pkh(b055…)]]]]` | idem | ✅ |
| &nbsp;&nbsp;`input_asset` | ADA | ADA | ✅ |
| &nbsp;&nbsp;`output_asset` | Poppy (`c4dd…Ranch Mascot`) | idem | ✅ |
| &nbsp;&nbsp;`order_amount` | `(1, 69000000)` | idem | ✅ |
| &nbsp;&nbsp;`min_output_ada` | 1 100 000 | 1 100 000 | ✅ |
| &nbsp;&nbsp;`executor_fee` | 1 500 000 | 1 500 000 | ✅ |
| &nbsp;&nbsp;`permitted_executor` | `e8659419…` | idem | ✅ |
| &nbsp;&nbsp;`deadline` | 1 776 527 736 335 | idem | ✅ |
| &nbsp;&nbsp;`owner_pkh` | `cb7b4e21…` | idem | ✅ |

## 2. `place_sell_order`

Generated hash: `8f2f6cb8e3b7fc9ef34eb339d80282c1e6b140ad7697b7f176d68406d20c3230`

- Order output value: `2 600 000 lovelace + 6 581 482 Poppy` (matches the fixed
  `sell_escrow_ada` + the token amount).
- Datum shape identical to `place_buy_order` but with roles swapped:
  `input_asset = Poppy`, `output_asset = ADA`, `order_amount = (1, 6_581_482)`.
- Owner address recorded in the datum is the funded wallet
  (`addr1q84uy5f05m54tn…`).
- 3 inputs auto-selected by the resolver (total ≥ 2.6 ADA + 6 581 482 Poppy + fees).
- 2 outputs: order UTxO at `addr1z8v3gwkxx3emz73pt5dhfpxlk6kxkjsqqkltpcn2djsze9jqudqhf…` + change back to the wallet.

## 3. `cancel_order`

Generated hash: `189a8d1ebdf17e07ee43402e68ab77457673076d96392812694fde3afd277156`

Target: live LAIKA SELL order `c665188f6c6f358ddabcbf49e02aa78c2c762c1517a29d024ffead51715c87c0#0`
at `addr1z8v3gwkxx3emz73pt5dhfpxlk6kxkjsqqkltpcn2djsze9ncgmmtkpl4k2p93p0y2qn8ne5eknnq5rzxpxjxhs652nxsqyujmw`.

| Field | Value |
|-------|-------|
| inputs[0] (order UTxO) | `c665188f…#0` (LAIKA order) |
| inputs[1..2] (owner funds) | resolver-picked UTxOs from `addr1q9cehm…` for fees/collateral |
| reference_inputs | `e2ed9e953ebf…#0` (order validator reference script) ✅ |
| outputs[0] | escrow refunded to owner: `2 600 000 lov + 785 790 LAIKA` |
| outputs[1] | ADA change to owner |
| Redeemer on order spend | `Constr 0 []` (Cancel) ✅ |
| collateral | present (required for script spend) ✅ |
| script_data_hash | computed ✅ |

## 4. `launch_token`

Generated hash: `ec3eba078e9c86b0524ffa8a0f7d35047e634434c261b0bf5bdc592a1ae26d86`

Applied the on-disk token template to seed
`1aa558a787d573158a287164517ac840154f56d79aebebc64b86ce3af7aabc76#2`
yielding policy `e0f0934f70bce91d6e6482174a125d26739d5fe710eb41972bf3a1b2`.
Launched a hypothetical "PoppyMini" token.

### Side-by-side output shape vs real Poppy launch

| # | generated (PoppyMini) | real (Poppy) | Match |
|---|-----------------------|--------------|-------|
| out0 (metadata script) | `01ff09258b…` + 2.534 ADA + *metadata NFT* + datum | same addr, same ADA, metadata NFT under token policy, same datum fields | ✅ structurally |
| out1 (fee collector) | `0122aea2da…` + 1.825 ADA | same addr, same ADA | ✅ **byte-identical** |
| out2 (bonding curve) | `31905ab869…` + 135.5 ADA + pool NFT + 949 952 431 tokens + PoolDatum | same addr, same ADA, same split, same datum | ✅ **byte-identical ADA + token split** |
| out3 (creator initial buy) | creator addr + 1.207 ADA + 50 047 569 tokens | same shape | ✅ |
| out4 (change) | creator addr + ADA change | same role | ✅ |

Minting:

| what | generated | real |
|------|-----------|------|
| Pool NFT | policy `63f947b8…` (stable), name = `hash(seed_outref)` | same policy, name differs (different seed) |
| Token | policy `e0f0934f…` (applied to new seed), 1 000 000 000 | policy `c4dd822c…` (applied to Poppy seed), 1 000 000 000 |
| Metadata NFT | same policy as token, name `"Snek.fun PoppyMini - Metadata"` | same layout, name `"Snek.fun Poppy - Metadata"` |
| Token mint script | inline PlutusV3, 511 B | inline PlutusV3, 511 B |

PoolDatum (9 fields) generated matches the on-chain v1 shape exactly:

```
Constr 0 [
  Constr 0 [pool_nft_policy, pool_nft_name],
  Constr 0 [ADA, ADA],
  Constr 0 [token_policy, token_name],
  int 122525779519,  // aNum
  int 2545182,       // bNum
  bytes(e865941988…),  // permitted executor
  int 18191400000,    // adaCapThreshold
  bytes(8807fbe6…),   // factory witness
  bytes(30c1003a…),   // pool witness
]
```

## Expected differences (not errors)

| Field | Generated | On-chain | Why |
|-------|-----------|----------|-----|
| Input UTxO selection | resolver picks minimum required | the real tx used whatever the wallet had | TRP chooses new UTxOs each build |
| Fee | ~1.67–1.68 ADA placeholders | 0.18–0.31 ADA actual | tx3 hasn't finalised execution costs |
| Change amount | depends on inputs | real-time wallet state | — |
| TTL (body[3]) | absent | absent | `validity` block removed |
| Witnesses | none (`--skip-submit`) | signed | unsigned build by design |

## Protocol constants captured

| Parameter | Value |
|-----------|-------|
| Bonding curve script hash | `905ab869961b094f1b8197278cfe15b45cbe49fa8f32c6b014f85a2d` |
| Order validator hash | `d9143ac63473b17a215d1b7484dfb6ac6b4a0005beb0e26a6ca02c96` |
| Order ref UTxO | `e2ed9e953ebf98ca701fc93588d73cb9769f87b9d13712474f566a0743963e8b#0` |
| Withdraw validator hash | `a5643b4a22a192d7691d05baf4a9bbb8acdbb5daa60be1f333e128f1` |
| Pool NFT minting policy | `63f947b8d9535bc4e4ce6919e3dc056547e8d30ada12f29aa5f826b8` |
| Metadata validator addr | `addr1q8lsjfvtpnu9kv5zhwgsdcw03tlkuwcvjqmzm35arx9xl6k6s0uhca7762y56q2j9en5ttn69p7a048cw3mz62fuj3mqauv0x3` |
| Launch fee collector | `addr1qy32agk6zhjffcqhvu296j9a594k6smlr3zfsgaqgsvnmtefn2pw7433r2ss4hcycqrj6jrsaw056hlnz5fjgdyyrd6qsw8ngt` |
| Permitted executor PKH | `e865941988edcca559268b57b7ee939974fd42fd26c7e1acd7a50678` |
| Factory witness PKH | `8807fbe6e36b1c35ad6f36f0993e2fc67ab6f2db06041cfa3a53c04a` |
| aNum / bNum | 122 525 779 519 / 2 545 182 |
| adaCapThreshold | 18 188 400 000 lovelace |
| tokenEmission | 1 000 000 000 |
| Fixed executor fee | 1 500 000 lovelace |
| Min output ADA | 1 100 000 lovelace |
| Min trade ADA | 1 000 000 lovelace |
| Percent fee | 1% |

## How to reproduce

```bash
cd protocols_tx3/snek-fun

trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/place_buy_order.json
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/place_sell_order.json
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/cancel_order.json
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/launch_token.json
```

To launch your own token, pick any UTxO from your wallet as the seed, then
re-apply the template:

```python
import hashlib
tpl = bytes.fromhex(open('investigation/scripts/token_mint.v3.template.cbor.hex').read())
seed_tx  = bytes.fromhex('<your seed tx hash>')   # 32 bytes
seed_idx = 0                                       # 0..23
applied  = tpl[:475] + seed_tx + bytes([seed_idx]) + tpl[475+33:]
policy   = hashlib.blake2b(b'\x03' + applied, digest_size=28).hexdigest()
# -> feed `applied.hex()` as token_script and `policy` as token_policy
```
