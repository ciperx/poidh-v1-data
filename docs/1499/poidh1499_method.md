# poidh Base v2 — per-wallet ledger (bounty #1499)

Snapshot block: **52,923,977** (Base mainnet, chain id 8453).
Files: `poidh1499_ledger.csv` (2,051 wallet rows) — columns `wallet,eth_paid_out,eth_earned,nft_held`.

## Contracts

| role | address |
|---|---|
| poidh v2 main contract | `0xb502c5856f7244dccdd0264a541cc25675353d39` |
| poidh v2 NFT (ERC-721) | `0xddfb1a53e7b73dba09f79fca24765c593d447a80` |

## Method

Read-only `eth_call` view enumeration only — no log scanning, no private keys, no writes.

1. **Bounties** — `bounties(uint256)` for ids `0..989` (`getBountiesLength() = 990`). Each record gives issuer, pot `amount`, and the accepted `claimId` (`0` = never completed).
2. **Claims** — `claims(uint256)` for ids `1..5319` (5,319 claims). `created_at == 1` is the contract's accepted flag; 531 claims are accepted and every accepted claim id matches the `claimId` stored on its bounty.
3. **Contributors** — `getParticipants(uint256)` at block 52,923,977 for every legitimately completed bounty. This returns the `(address[], uint256[])` contribution arrays maintained by the contract for open/crowdfunded bounties. For solo bounties the array is empty and the issuer funded the whole pot. The participant amounts sum exactly to `bounties[id].amount` for all 498 included bounties (0 mismatches).
4. **NFTs** — `ownerOf(uint256)` for token ids `0..5319` at block 52,923,977, plus `totalSupply()`.

Batched through Multicall3 (`0xcA11bde05977b3631167028862bE2a173976CA11`) as `aggregate3` `eth_call`s.

### Column definitions

- `wallet` — lowercase EVM address.
- `eth_paid_out` — ETH this wallet contributed to pots that were **legitimately completed** (decimal ETH, 18 dp). Solo-bounty pots are credited to the issuer; crowdfunded/open-bounty pots are attributed pro-rata to each participant's on-chain contribution. Contributions to bounties that were cancelled, refunded, still open, or fraudulently completed are **not** counted.
- `eth_earned` — gross ETH earned from accepted claims on legitimately completed bounties, **before** the 2.5% protocol fee (a 1 ETH bounty credits 1 ETH).
- `nft_held` — number of poidh v2 NFTs currently held at the snapshot block. NFTs still held by the protocol contract (`0xb502c5…3d39`) are **not** attributed to any wallet; they are reported separately below.

Rows are the union of all bounty issuers (990), all claim submitters (5,319 claims), all NFT holders, and all contributors — 2,051 unique wallets, including wallets whose cells are all zero.

## December 8, 2025 exploit exclusion

- Exploit tx: `0x0545a4e5800632ba2194fb9349264ff7f3d3bb18d28ee168d57369b14422f11f`, block **39,216,925**, timestamp **1765223197**.
- Attacker EOA: `0xc9cade0aa7bd77b00d1ab057a3684c698725cf54`; the fraudulent claims were submitted by contract `0xcb18f71e12b85058974f181e8cca769ab5eedbbd`.
- Exploit claims: ids **5275–5310** (36 claims submitted at the exploit timestamp, the same ids as the 69 NFT transfer events in that tx). 33 were accepted, 3 (5297–5299) were not.
- Excluded: the **33 completed bounties** whose accepted claim is one of 5275–5310. Their pots (`0.175085 ETH` gross) are removed from both `eth_paid_out` and `eth_earned`; the contributing wallets get no credit for funding them, and the fraudulent claimer gets no earnings. The other 3 exploit claims never completed a bounty, so those bounties were already excluded as uncompleted.
- Excluded bounty ids: 50, 59, 247, 460, 534, 566, 576, 668, 687, 732, 736, 737, 739, 741, 742, 744, 746, 748, 793, 812, 817, 833, 835, 872, 873, 874, 875, 876, 924, 931, 960, 969, 976.
- Whitehat recovery transactions and user refunds are balance transfers, not bounty completions, so they never enter `eth_earned` or `eth_paid_out`. Cancelled/refunded bounties carry `claimId = 0` and are likewise excluded.
- Sanity: the attacker address and the exploit-claim contract both have `eth_earned = 0` and `eth_paid_out = 0` in the CSV.

Note on the NFT column: the 36 exploit-minted tokens (5275–5310) are real ERC-721 tokens and are included in `nft_held` according to current on-chain ownership (33 held by 16 wallets, 3 held by the protocol contract). If a strict "legitimate completions only" NFT view is wanted, subtract those 33 wallet-held tokens, giving 499 circulating legitimate NFTs.

## Reconciliation

| item | value |
|---|---|
| rows (unique wallets) | **2,051** |
| bounties enumerated | 990 |
| completed bounties (`claimId > 0`) | 531 |
| exploit completions excluded | 33 |
| legitimately completed bounties | **498** |
| accepted claims (all) | 531 |
| accepted claims excluded (exploit) | 33 |
| `sum(eth_paid_out)` | **3.985555687911831783 ETH** (`3985555687911831783` wei) |
| `sum(eth_earned)` | **3.985555687911831783 ETH** (`3985555687911831783` wei) |
| `sum(nft_held)` (wallets) | **532** |
| NFTs held by protocol contract | **4,788** |
| NFT `totalSupply` at block 52,923,977 | **5,320** (532 + 4,788 = 5,320) |
| wallets with non-zero paid / earned / NFT | 256 / 299 / 191 |

Because every included bounty is attributed to its contributors for exactly its gross pot, `sum(eth_paid_out)` and `sum(eth_earned)` are equal by construction; the independent re-derivation matches both, and `sum(earned)` equals the gross total of the 498 included bounty pots.

Reproduce: `poidh1499_build.py` (inputs: `poidh_v2_enum.json`, `poidh_v2_claims_all.json`, `exploit_tx.json`, NFT owner snapshot at block 52,923,977) and `poidh1499_verify.py` (independent re-derivation; all checks pass).
