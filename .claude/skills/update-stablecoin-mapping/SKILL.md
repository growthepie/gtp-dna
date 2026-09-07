---
name: update-stablecoin-mapping
description: Add a new stablecoin or new addresses for an existing stablecoin in stables/stables_config_v2.py. Use this skill whenever the user wants to add a new stablecoin token, add addresses for an existing token on additional chains, a chain already exists in address_mapping and needs new tokens added, wants to verify or audit which stablecoins are mapped for a specific chain, or the user says anything like "add stablecoin", "new token", "add token", "missing stablecoin", "add addresses", "double check", "check stablecoin mapping", "verify stablecoins", or "all stablecoins mapped" in the context of stablecoin tracking.
---

You are helping update `stables/stables_config_v2.py` — either by adding a new stablecoin to `coin_mapping`, adding `address_mapping` entries for an existing token on new chains, or both.

## Step 1 — Understand what is being added

Clarify the task (if not already clear):

- **New stablecoin token**: needs a new `coin_mapping` entry + one or more `address_mapping` entries per chain.
- **New chain addresses for an existing token**: the `token_id` already exists in `coin_mapping`, just missing from `address_mapping` for specific chains.

> If the **entire chain** is missing from `address_mapping`, suggest the `add-chain-stablecoin-mapping` skill instead — it handles full chain onboarding including all stablecoins at once.

Read `stables/stables_config_v2.py` to check the current state before proceeding.

**T3 Rules — check before adding anything:**
1. No value-increasing stablecoins (e.g. sUSDS, aTokens, yield-bearing wrappers).
2. No stablecoins that mainly wrap another stablecoin, unless it is a bridge representation.
3. Only stablecoins that anyone can own (no permissioned/institutional-only tokens like BUIDL).

If the token violates any rule, stop and tell the user.

## Step 2 — Gather token metadata (new stablecoin only)

Skip this step if adding addresses for an already-existing `token_id`.

### 2a — Basic fields

Collect or derive the following:

| Field | Description |
|---|---|
| `symbol` | Token symbol (e.g. `USDC`, `FRXUSD`) |
| `coingecko_id` | List of CoinGecko IDs for this token. Look up on coingecko.com. |
| `metric_key` | `"direct"` if natively issued; `"bridged"` if it represents a bridged canonical token |
| `bridged_origin_chain` | If `metric_key == "bridged"`: chain where the canonical token lives (e.g. `"ethereum"`) |
| `bridged_origin_token_id` | If `metric_key == "bridged"`: the `token_id` of the origin token in `coin_mapping` |
| `fiat` | Peg currency: `usd`, `eur`, `gbp`, `chf`, `aud`, `jpy`, etc. Must exist in [fiat.json](https://raw.githubusercontent.com/growthepie/gtp-frontend/a570d6bf4856df66734eb6b4ddc6e2cdc164a71f/public/dicts/fiat.json) |
| `logo` | CoinGecko large image URL |
| `color_hex` | Brand color for charts (best guess from logo) |

Signals that a token is bridged: symbol suffix like `.e`, `(bridged)` in the name, a CoinGecko ID containing "bridged", or it arrived via a canonical bridge.

### 2b — Resolve `owner_project`

The `owner_project` field must match a slug in the growthepie OSO-directory. **Always run the lookup script** before proposing a new entry:

```bash
python .claude/skills/add-chain-stablecoin-mapping/scripts/fetch_owner_project.py <keyword1> [keyword2 ...]
```

Use the token symbol, issuer name, or project name as keywords (e.g. `lumi luausd`, `circle usdc`).

- If one or more matches are returned, pick the most relevant slug and show it to the user for confirmation.
- **If no match is found**, set `owner_project` to `null` and **explicitly warn the user** that this field must be filled in before merging. They may need to create a new entry at https://www.openlabelsinitiative.org/project. The full directory is at `https://api.growthepie.com/v1/labels/projects.json`.

### 2c — Derive `token_id`

```
token_id = "{owner_project}_{symbol_lowercase}"
```

If `owner_project` is null, use a placeholder like `"tbd_{symbol_lowercase}"` and flag it for the user.

## Step 3 — Find addresses per chain

For each chain where the stablecoin is deployed, find the **local token address** (not a bridge escrow) and **decimals**.

There are two scenarios — pick based on what the user specified:

---

### Scenario A — Adding a token to one specific chain (most common)

First, get the chain's aliases:

```bash
python .claude/skills/add-chain-stablecoin-mapping/scripts/get_chain_info.py <origin_key>
```

Then run all four discovery scripts **in parallel** — they are independent read-only fetches:

```bash
python .claude/skills/add-chain-stablecoin-mapping/scripts/fetch_l2beat_tvs.py <aliases_l2beat>
python .claude/skills/add-chain-stablecoin-mapping/scripts/fetch_defillama_assets.py <aliases_defillama>
python .claude/skills/add-chain-stablecoin-mapping/scripts/fetch_coingecko_ecosystem.py <aliases_coingecko_chain>
python .claude/skills/add-chain-stablecoin-mapping/scripts/fetch_dune.py --chain <aliases_dune_chain> --symbol <symbol>
```

The `aliases_dune_chain` is the Dune chain identifier (e.g. `ethereum`, `base`, `arbitrum`) — typically the lowercase `origin_key` or a close variant.

Filter all results by matching symbol. Merge the four outputs:
- Where two or more sources agree on an address → highly reliable, use it.
- Single-source results → flag for manual verification.

Script-specific notes:
- **`fetch_l2beat_tvs.py`**: returns `native` (address ready to use) and `bridged_via_l1_escrow` (no local L2 address — look up via CoinGecko or block explorer). If the script errors with 404, find the correct slug at `https://api.github.com/repos/l2beat/l2beat/contents/packages/config/src/tvs/json`.
- **`fetch_defillama_assets.py`**: returns curated `{ symbol, address }` pairs. If it errors with "Chain not found", check `aliases_defillama` from `get_chain_info.py`.
- **`fetch_coingecko_ecosystem.py`**: returns stablecoins in the chain's ecosystem category with address and decimals. If it errors with "Category not found", the chain may use a different CoinGecko ecosystem slug.
- **`fetch_dune.py`**: queries Dune Analytics (query 6910797) for on-chain stablecoin data filtered by chain and symbol. Requires `DUNE_API_KEY`. Returns address and decimals per chain.

---

### Scenario B — Finding all chains a token is deployed on

Run both discovery scripts **in parallel**:

```bash
python .claude/skills/update-stablecoin-mapping/scripts/fetch_coingecko_token.py <coingecko_id>
python .claude/skills/add-chain-stablecoin-mapping/scripts/fetch_dune.py --symbol <symbol>
```

`fetch_coingecko_token.py` returns the token's symbol, name, logo, and a `deployments` list — one entry per chain — with `platform` (the `aliases_coingecko_chain` key), `address`, and `decimals`.

`fetch_dune.py` returns rows from Dune query 6910797 filtered by symbol across all chains. Use this to cross-validate CoinGecko results and discover chains that CoinGecko may be missing. Requires `DUNE_API_KEY`.

To map each result back to a growthepie `origin_key`: compare chain names against `aliases_coingecko_chain` and `aliases_dune_chain` from `get_chain_info.py <origin_key>` for chains already in `address_mapping`. Only add entries for chains already present in `address_mapping` (use `add-chain-stablecoin-mapping` for new chains).

---

### Fallback — Manual lookup

If scripts return no data for a specific chain:
- `https://defillama.com/stablecoins/{aliases_defillama}` — lists stablecoins with addresses
- The chain's block explorer — search by token symbol

#### Celo authoritative registry

For `origin_key == "celo"`, also audit Celo's official stablecoin documentation:

- `https://docs.celo.org/build-on-celo/build-with-local-stablecoin`

Use its **Celo Mainnet** contract addresses, not Celo Sepolia testnet addresses. Cross-check new or conflicting entries against the issuer's current deployment documentation and CeloScan before proposing a change. Treat the page's GoodDollar `G$` entry as a UBI token rather than a fiat-pegged stablecoin unless the repository's eligibility rules change.

## Step 4 — Verify addresses

- For `metric_key == "bridged"` tokens: confirm the address is the bridged token on the L2 (not the L1 escrow). Verify `bridged_origin_chain` and `bridged_origin_token_id` are correct — these prevent double-counting supply.
- For `metric_key == "direct"` tokens: confirm the token is natively issued on this chain.
- Do **not** remove existing entries in `address_mapping`. Only flag conflicts (same `token_id`, different address or decimals) and ask the user how to resolve.

## Step 5 — Show proposed changes and confirm

Present exactly what will be written.

**New `coin_mapping` entry (if new token):**
```python
{
    "owner_project": "<slug or null>",        # from OSO-directory lookup
    "token_id": "<owner_project>_<symbol>",
    "symbol": "SYMBOL",
    "coingecko_id": ["coingecko-id"],
    "metric_key": "direct",                   # or "bridged"
    "bridged_origin_chain": None,             # or "ethereum"
    "bridged_origin_token_id": None,          # or "circlefin_usdc"
    "fiat": "usd",
    "logo": "https://...",
    "color_hex": "#XXXXXX"
}
```

**New `address_mapping` entries:**
```python
# Under address_mapping["<origin_key>"]:
"<token_id>": {
    "address": "0x...",    # local token address on this chain, lowercase
    "decimals": 6,
    "exclude_balances": [  # optional: addresses to exclude from supply (e.g. treasury, reserves)
        "0x..."
    ]
}
```

List any chains where the address could not be found or verified — the user may want to add those manually later.

Ask the user to confirm before writing anything.

## Step 6 — Write the changes

1. **`coin_mapping`** (new token only): append the new dict at the end of the `coin_mapping` list, before the closing `]`. Match existing formatting (4-space indent).
2. **`address_mapping`** (new chain addresses): **always** look up the exact line range first, then insert into the correct chain block:

```bash
python .claude/skills/update-stablecoin-mapping/scripts/get_address_mapping_lines.py <origin_key>
```

This returns:
```json
{
  "polygon_pos": {
    "block_start": 2728,
    "last_entry_end": 2897,
    "block_end": 2898
  }
}
```

Read from `last_entry_end - 5` to `block_end` to see the final token entry, then insert the new entry after it (before the closing `}`). **Never rely on string-matching a token address or token_id to locate the insertion point** — the same token_id (e.g. `tetherto_usdte`) appears in many chains and will match the wrong block. Always use the line numbers from this script.

Keep addresses lowercase. Do not add comments inside `address_mapping`.

## Step 7 — Done

After writing, confirm success and remind the user:
- If `owner_project` is `null`, it must be filled in before merging — suggest https://www.openlabelsinitiative.org/project to register the project.
- If `coingecko_id` is set, the discovery scripts can find new chain addresses automatically in the future.
- Open a PR if this is a community contribution.
