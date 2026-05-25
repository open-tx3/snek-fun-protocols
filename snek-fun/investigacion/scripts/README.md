# snek.fun on-chain scripts (mainnet v1)

Extracted from real launch transactions via Koios `tx_cbor`.
See ../snek-fun-research.md for transaction references.

| File | Plutus ver. | Bytes | Policy id / script hash | Stability |
|------|-------------|-------|-------------------------|-----------|
| `pool_nft_policy.v2.cbor.hex` | V2 | 575 | `63f947b8d9535bc4e4ce6919e3dc056547e8d30ada12f29aa5f826b8` | **stable across every launch** |
| `token_mint.v3.template.cbor.hex` | V3 | 511 | — template only | 32-byte hole at offset 475..506 |
| `token_mint.v3.poppy.cbor.hex` | V3 | 511 | `c4dd822c92c486a06933042c49e0f4e0a0440907420d9743321f4105` | sample (Poppy, applied) |

## How we derived the token template

Cross-compared 5 mainnet launches (KING, SNOP, S, RETRO, BBCRAW, plus the
original Poppy and Charlo). Every one produced a 511-byte PlutusV3 script.

| pair                   | prefix | suffix | middle |
|------------------------|--------|--------|--------|
| KING vs SNOP (idx 0)   | 475 B  | **4 B** | 32 B (tx_hash only) |
| KING vs S     (idx 1)  | 475 B  | **3 B** | 33 B (tx_hash + idx byte) |
| KING vs RETRO (idx 2)  | 475 B  | **3 B** | 33 B |
| KING vs BBCRAW (idx 2) | 475 B  | **3 B** | 33 B |

So the applied-parameter window is **33 bytes at offset 475..507**:

- bytes 475..506 — `seed_outref.tx_hash` (32 B)
- byte 507       — `seed_outref.output_index` as a single CBOR int byte
  (only valid when `index ≤ 23`, which covers virtually every real wallet input)

Bytes 508..510 are the stable 3-byte tail of the compiled program.

## Computing the policy id

Cardano script hash = `blake2b_224(tag_byte || script_bytes)`, where:

- `tag = 0x02` for PlutusV2 (pool NFT)
- `tag = 0x03` for PlutusV3 (token)

Example:

```python
import hashlib
script = bytes.fromhex(open('token_mint.v3.poppy.cbor.hex').read())
policy_id = hashlib.blake2b(bytes([0x03]) + script, digest_size=28).hexdigest()
# -> c4dd822c92c486a06933042c49e0f4e0a0440907420d9743321f4105
```

## Applying the template for a new launch

```python
import hashlib
template = bytes.fromhex(open('token_mint.v3.template.cbor.hex').read())
assert len(template) == 511

seed_tx_hash = bytes.fromhex('...')  # seed input's tx hash (32 bytes)
seed_index   = 0                     # seed input's output index (0..23)
assert len(seed_tx_hash) == 32 and 0 <= seed_index <= 23

applied = (
    template[:475]
    + seed_tx_hash
    + bytes([seed_index])            # 1-byte CBOR int
    + template[475+33:]
)
policy_id = hashlib.blake2b(bytes([0x03]) + applied, digest_size=28).hexdigest()
```

Round-trip check (Poppy): `seed_tx_hash=e395…2ef1`, `seed_index=0` →
`policy_id = c4dd822c92c486a06933042c49e0f4e0a0440907420d9743321f4105` ✓.

Feed `applied.hex()` as `token_script` and `policy_id` as `token_policy` into
`invoke-args/launch_token.json`.
