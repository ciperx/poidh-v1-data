# poidh v1 (Arbitrum) — per-wallet dataset

Main contract `0xdffe8a4a4103f968ffd61fd082d08c41dcf9b940`, NFT contract `0xdfFE8A4a4103f968Ffd61fd082D08c41dCf9b940` (Arbitrum One).

`poidh_v1_wallets.csv` — one row per wallet that ever interacted with poidh v1:

| column | meaning |
|---|---|
| `wallet` | address (lowercase) |
| `eth_paid_out_wei` | ETH this wallet actually paid out for completed bounties (funded pots that were accepted), 2.5% protocol fee not subtracted |
| `eth_earned_wei` | ETH earned cumulatively from accepted claims, gross (a 1 ETH bounty credits 1 ETH) |
| `nfts_held` | poidh v1 NFTs currently held by the wallet |

## Totals

- wallets: 414 (210 with a non-zero cell)
- sum(eth_paid_out_wei) = sum(eth_earned_wei) = 3_726_923_002_000_000_000 wei (3.726923002 ETH)
- sum(nfts_held) = 278
- NFT supply cross-check: totalSupply = 722, of which 444 are held by the protocol contract, 278 by wallets — matches the column above.

## Method

Read-only `eth_call` view enumeration (no log scanning, no keys): `bounties(0..495)`, `claims(0..721)`, `tokenByIndex(0..totalSupply-1)`, plus `eth_getLogs` over 10M-block windows (3,467 logs) used only to catch wallets that funded or claimed and were otherwise missed. Block 517066757 snapshot.

Reproduce: `build.py` (enumerate → decode → CSV) and `verify.py` (independent re-derivation of every total).
