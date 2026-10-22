# poidh v2 on Arbitrum One — wallet-level ETH flows & NFT holdings

**Dataset:** `poidh_v2_arbitrum.csv` (104 wallets) · **Detail:** `poidh_v2_arbitrum_bounties.csv` (180 bounties) · **Totals:** `poidh_v2_arbitrum_totals.json`

All figures below are read directly from Arbitrum One state at **block 517,097,828** with `eth_call` only — no indexer, no explorer, no `eth_getLogs`.

| Item | Value |
|---|---|
| Bounty contract | `0x0aa50ce0d724cc28f8f7af4630c32377b4d5c27d` (PoidhV2, verified source matched) |
| NFT contract | `0xddfb1a53e7b73dba09f79fca24765c593d447a80` (POIDH V2) |
| Treasury | `0x1a706a29822921d6c641e472bd33cf945a13fef9` |
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` |
| Bounties | 180 (112 completed, 24 open, 44 cancelled/refunded) |
| Claims (NFTs minted) | 437 |
| **Σ ETH paid out** | **0.31808279 ETH** |
| **Σ ETH earned (gross)** | **0.31808279 ETH** |
| Protocol fee (2.5%) | 0.00795206975 ETH |
| Net to claimers | 0.31013072025 ETH |
| NFTs held by wallets | 112 (325 still held by the bounty contract) |

## 1. Enumeration

1. `getBountiesLength()` → **180**; `bountyCounter()` = 180; `claimCounter()` = 437; NFT `totalSupply()` = 437.
2. `bounties(i)` for i ∈ [0, 180) — 6 batched `aggregate3` calls, 30 sub-calls each. Returns
   `(id, issuer, name, description, amount, claimer, createdAt, claimId)`.
3. `getParticipants(i)` for i ∈ [0, 180) — 8 batched calls, 25 sub-calls each. Returns
   `(address[] participants, uint256[] participantAmounts)`.
4. `getClaimsByBountyId(i)` for i ∈ [0, 180) — 15 batched calls, 12 sub-calls each. Returns
   `Claim[]` = `(id, issuer, bountyId, bountyIssuer, name, description, createdAt, accepted)`.
5. `ownerOf(tokenId)` on the NFT contract for every tokenId ∈ [0, 437) — 8 batched calls, 60 each.

`aggregate3((address,bool,bytes)[])` = `0x82ad56cb`; sub-calls run with `allowFailure=true` and every
result is asserted `success == true` (a reverted sub-call aborts the run rather than being dropped).

Every call in steps 1–5 is pinned to the same block, so the snapshot is internally consistent.

## 2. Definitions (and why they are what they are)

The deployed contract was checked against its verified source (`PoidhV2.sol`, Sourcify full match),
not assumed from the Base ABI: on Arbitrum the claim struct ends in `bool accepted`, and the
crowdfund path (`joinOpenBounty` / `withdrawFromOpenBounty` / `participantAmounts`) exists. The Base
ABI in `poidh_v2_abi.json` has neither, so it cannot enumerate this deployment.

**Completed bounty** — `claimer != 0x0` **and** `claimer != issuer` **and** the linked claim
(`bounties(i).claimId`) has `accepted == true` with `claim.issuer == bounty.claimer`.
All 112 completed bounties pass that check; all 44 refunds are excluded by the `claimer == issuer`
rule (`cancelSoloBounty` / `cancelOpenBounty` set `claimer = msg.sender = issuer` and leave `amount`
stale), and the 24 open bounties have `claimer == 0x0`.

**`eth_paid_out`** — the pot of every *completed* bounty attributed to whoever actually put the ETH in:
- open/crowdfunded bounties: the on-chain `participantAmounts[]` split (issuer is `participants[0]`;
  a participant who withdrew is zeroed on-chain, so withdrawn money is correctly not counted);
- solo bounties (empty participant list): the whole `amount` to the issuer.
For all 180 bounties the contributions sum exactly to `bounties(i).amount` (0 mismatches), including
the 61 completed bounties that were crowdfunded. 3 wallets paid in without ever creating a bounty
(0.0555 ETH of the total) — they are ordinary crowdfund contributors.

**`eth_earned`** — gross, before the protocol fee. `_acceptClaim` computes
`fee = amount * 25 / 1000` and `payout = amount - fee` and sends both in the same transaction; it does
**not** zero `amount`, so `bounties(i).amount` is the gross pot at settlement. Hence
`earned = amount = fee * 40` exactly — verified for all 112 rows (`fee*40 == amount`, 0 rounding
cases, because every pot is divisible by 40 wei).

**`nft_count`** — **current holdings**, not mints. A poidh v2 NFT is minted to the bounty contract with
`tokenId == claimId` and transferred on acceptance to the bounty issuer (102 of 112; 2 went to the
claimer, 8 were later transferred on). `ownerOf` is read for all 437 tokens, so secondary transfers
are reflected; 325 unaccepted claim NFTs remain in the contract and are not attributed to any wallet.

## 3. Reconciliation

| Check | Result |
|---|---|
| Σ `eth_paid_out` == Σ `eth_earned` == Σ amount of the 112 completed bounties | ✅ 318,082,790,000,000,000 wei each |
| Σ fee == Σ `amount*25//1000`; fee × 40 == Σ earned | ✅ 7,952,069,750,000,000 wei; ×40 = earned |
| Σ net payout == Σ earned − Σ fee | ✅ 310,130,720,250,000,000 wei |
| Σ `bounties_created` over wallets == 180 | ✅ |
| Completed pot + cancelled refunds + open escrow == Σ all 180 `amount` | ✅ 0.31808279 + 0.24255493 + 0.02485801 = 0.58549573 ETH |
| NFTs: 112 wallet-held + 325 contract-held == `totalSupply` 437 | ✅ |
| Per-wallet earned == Σ its won bounty rows; per-wallet paid == Σ its contributor entries | ✅ (recomputed from the detail CSV) |
| 18 random completed bounties + 12 random NFTs re-read by plain `eth_call` (no multicall, 2 RPC hosts) | ✅ 0 failures |

**One discrepancy in the contract's own books, reported rather than smoothed over:** the bounty
contract holds **0 ETH** while its 24 open bounties still carry **0.02485801 ETH** of unclaimed
`amount`; a simulated `withdrawFromOpenBounty()` from a non-issuer participant reverts
`transferFailed()` (`0xe465903e`). Settled flows are unaffected — `_acceptClaim` requires
`amount <= address(this).balance` and pays out atomically, so every one of the 112 completed payouts
did happen. This dataset does not attribute the missing open escrow to any wallet.

## 4. Files & reproduction

| File | sha256 |
|---|---|
| `poidh_v2_arbitrum.csv` | `c9640cca949c396cb2d81e7afe942e7247376ea4fabe4ae53848463a9af362f1` |
| `poidh_v2_arbitrum_bounties.csv` | `66f01d4461427dd41b8952db305c38e12f10832c067b92e4e8bd97ee43a10c94` |
| `poidh_v2_arbitrum_totals.json` | `ad3b30e160e86d140d5e8b31cfcbd50ae6a55285d20fe8fe5337657603559fdb` |
| `poidh_v2_arb_raw.json` (raw snapshot) | `847cac98b2d3ddef3aa9811bb17807358825ec6d6b5351788d81f606775e7ac6` |

```bash
/opt/data/.venv-poidh/bin/python poidh_v2_arb_enumerate.py   # 180 bounties, 437 claims, 437 NFTs -> raw json
/opt/data/.venv-poidh/bin/python poidh_v2_arb_build.py       # raw json -> wallet CSV, bounty CSV, totals
/opt/data/.venv-poidh/bin/python poidh_v2_arb_verify.py      # independent recompute + on-chain spot check
```

RPCs (all three answered; the run used the first): `https://arb1.arbitrum.io/rpc`,
`https://arbitrum.drpc.org`, `https://arbitrum-one.publicnode.com`. Any of them reproduces the same
snapshot at block 517,097,828; later blocks only add bounties/claims, which shifts totals but not the
method.

## 5. Column dictionary (`poidh_v2_arbitrum.csv`)

`wallet` · `bounties_created` (all bounties issued, incl. cancelled) · `eth_paid_out` (ETH this wallet
put into bounties that were later paid out) · `eth_earned` (gross ETH received as the accepted claimer)
· `nft_count` (poidh v2 NFTs currently held) · `bounties_completed` (issued and settled) ·
`bounties_won` (accepted claims) · `claims_submitted` · `eth_earned_net` (after the 2.5% fee) ·
`eth_fee_paid` · `eth_paid_out_wei` / `eth_earned_wei` / `eth_fee_paid_wei` (exact integers) ·
`nft_ids` (tokenIds held, `;`-separated).

ETH strings are exact 18-decimal renderings of the integer wei columns — no float rounding anywhere.
