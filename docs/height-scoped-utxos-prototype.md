# Height-scoped transparent UTXO queries (Tier 1 prototype)

> Status: prototype on branch `claude/tender-cannon-lg9unv`. Not upstreamed.
> No DB migration; reuses the existing `utxo_by_out_loc` column family.

## Problem

`getaddressutxos` is address-scoped with no height filter: it returns each
address's *full current* UTXO set on every call. An indexer tracking ~100k
addresses therefore issues ~1,000 RPCs per block, every ~75s, re-fetching almost
entirely unchanged state. The cost is `O(addresses ever exposed)` rather than
`O(what actually changed)`.

The address indexes are the reason: `utxo_loc_by_transparent_addr_loc` is keyed
`[AddressLocation][OutputLocation]` — **address is the prefix**, so the only
efficient question it answers is "all UTXOs for address X". There is no
height-prefixed index, so "what changed since height H, across my addresses"
forces a per-address rescan.

## Key idea

Query an index where **height is the prefix**. Zebra already has one:
`utxo_by_out_loc`, keyed by `OutputLocation` (3-byte big-endian height → tx index
→ output index), is the *live* unspent set (inserted on creation, deleted on
spend). A single ordered range scan over `[H, tip]` returns exactly the outputs
**created at/after H that are still unspent now** — `O(outputs in the window)`,
i.e. `O(what changed)` — with no schema change. Each scanned output is filtered
to the requested address set via `transparent::Output::address(network)`.

## What this prototype adds

State layer:
- `ZebraDb::finalized_address_utxos_in_height_range(network, addresses, height_range)`
  — the height-ordered range scan with address filtering.
  (`zebra-state/src/service/finalized_state/zebra_db/transparent.rs`)
- `read::address_utxos_in_height_range(...)` — mirrors `address_utxos`,
  swapping the finalized query for the range scan and filtering the
  non-finalized creations to the height window. Reuses the existing
  finalized/non-finalized consistency + retry logic.
  (`zebra-state/src/service/read/address/utxo.rs`)
- `ReadRequest::UtxosByAddressesInHeightRange { addresses, height_range }`
  → `ReadResponse::AddressUtxos` (`zebra-state/src/request.rs`, `service.rs`)

RPC layer:
- New method `getaddressutxosbyheight` (a Zebra extension; no zcashd
  equivalent — kept separate so `getaddressutxos`'s zcashd compatibility is
  untouched). Params: `addresses`, optional inclusive `start_height` /
  `end_height` (default full range). (`zebra-rpc/src/methods.rs`)

## Semantics and limitations (read before using)

- Returns outputs **created at/after `start_height`** that are **still unspent**
  at the queried tip. It is the *creation* side of a changeset.
- It does **not** report outputs created *before* `start_height` that were
  **spent** within the range — those are already gone from `utxo_by_out_loc`.
  Full incremental UTXO-set maintenance (the recurring poll) needs a
  **spend-by-height index** (Tier 2): a new column family keyed by *spend*
  height, written where `utxo_by_out_loc` is deleted, plus a DB format bump and
  a backfill migration. Out of scope for this prototype.
- An empty `addresses` set matches nothing (same as `getaddressutxos`).
  A general "scan the whole UTXO set" (match-all) mode would also need the
  non-finalized side to enumerate all addresses; not wired here.

## Where it helps today

- **Incremental polling** with a non-zero `start_height` (last-synced height):
  the scan window is tiny (one or a few blocks), so this is `O(changes)` for the
  *creation* side, replacing the bulk of the per-address re-fetch.
- **Initial import** of a large/dense address set with the full range: one
  sequential scan + one round-trip instead of N per-address indexed lookups +
  N round-trips. Wins when the address set is large relative to the chain's
  UTXO set; loses for a handful of sparse addresses (the per-address index is
  better there). Import is a one-time cost either way.
- For import at scale the response should be **paginated by a height cursor**
  (same cursor as incremental polling) — noted as a follow-up.
