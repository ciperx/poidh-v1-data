# poidh v1 per-wallet ledger (Arbitrum)

Derived **only from on-chain logs** of the poidh v1 contract on Arbitrum One:
`0xdfFE8A4a4103f968Ffd61fd082D08c41dCf9b940` (this address is both the bounty
contract and the ERC-721 claim NFT contract).

* Block range scanned: 100,000,000 - 519,999,999 (full history; 42 x 10M-block
  `eth_getLogs` windows, endpoint `https://arb1.arbitrum.io/rpc`).
* Logs collected: 3467
* Rows: 414 wallets that ever interacted with v1.
* No indexer was used, no third-party CSV was copied.

## Event decoding (topic0 verified by keccak of the signature)

| topic0 | event | count |
|---|---|---|
| `0xd00fa077...` | `BountyCreated(uint256 id, address indexed issuer, string name, string description, uint256 amount, uint256 createdAt)` | 496 |
| `0xad535e15...` | `ClaimCreated(...)` | 722 |
| `0x68669d60...` | `ClaimAccepted(uint256 bountyId, uint256 claimId, address claimIssuer, address bountyIssuer, uint256 fee)` | 278 |
| `0xbe34663b...` | `BountyCancelled(uint256 bountyId, address issuer)` | 171 |
| `0xddf252ad...` | ERC-721 `Transfer(address,address,uint256)` | 1073 |

## Column definitions

* `eth_paid_out_wei` - ETH actually transferred out of the contract to that
  wallet for completed bounties (`bountyAmount - fee`, from the
  `ClaimAccepted` event; the protocol fee is therefore excluded).
* `eth_earned_wei` - gross value of bounties won, **2.5% fee not deducted**
  (a 1 ETH bounty is credited as 1 ETH), taken from `BountyCreated.amount`.
* `nfts_held` - poidh v1 claim NFTs **currently** held, rebuilt from the full
  ERC-721 `Transfer` stream (mint -> current owner; burned tokens removed).

## Reconciliation

* `sum(eth_earned_wei)`  = 3726923002000000000 wei = 3.726923002 ETH
* `sum(eth_paid_out_wei)` = 3633749926950000000 wei = 3.633749927 ETH
* difference = 93173075050000000 wei = 0.093173075 ETH
* exactly 2.5% of gross: 93173075050000000 wei -> matches to the wei for all 278
  accepted claims (checked claim-by-claim: `gross - fee == fee * 39` is true for
  every one of the 278 `ClaimAccepted` events).
* `sum(nfts_held)` = 722 tokens across 80 wallets
  (722 tokens ever minted, 1073 transfer events).

## Reproduce

`build.py` re-runs the whole pipeline: fetch logs, decode events, write CSV.
Requires `pycryptodome` (topic0 verification) and nothing else.

Data is read at the current chain head minus reorg safety; block range is fully
historical so the result is final.
