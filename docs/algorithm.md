# Detection algorithm

The detector targets one pattern: a new wallet that suddenly puts a lot of money into one to three related markets close to their end, where the outcome can be influenced by a single person or organization.

## Stage 1: manipulability filter

A market passes only if all of these hold:

- It has a clear outcome maker: either `negRisk = true`, or the question contains a decision verb (`pardon`, `strike`, `launch`, `name`, `announce`, plus `search`, `streamed`, `elect`, `win`).
- Liquidity is under $100k. Popular markets are hard for one person to move.
- It has an end date, which the timing signal needs.

About 30% of markets pass (down from 97% under an earlier, looser filter). All five in-window known insiders' markets pass. Code: `src/detection/manipulability.ts`.

## Stage 2: per (wallet, market) score

For each (wallet, token_id) pair with at least $100 traded in a manipulable market:

| Signal | Formula | Scoring |
|---|---|---|
| E1 wallet age | days from first USDC.e deposit to first trade in this market | under 1 day 1.0, under 7 days 0.9, under 30 days 0.6 |
| E2 trade size | `max(biggest_single_trade, market_total / 5)` | $50k+ 1.0, $10k+ 0.8, $5k+ 0.6, $1k+ 0.4 |
| E3 entry timing | hours before resolution at first trade | under 24h 1.0, under 72h 0.9, under 1 week 0.7, under 1 month 0.5, under 3 months 0.3 |
| E4 total in market | sum of the wallet's USDC into this market | $100k+ 1.0, $25k+ 0.8, $5k+ 0.5, $1k+ 0.3 |

Pair score = `MAX(avg(E1..E4), avg of top 3 of E1..E4) * 100`.

The top 3 of 4 rule catches wallets with one weak signal. The Trump pardon CZ wallet bet three months before resolution (weak timing) but was fresh, concentrated and large. A plain average scores it 45; top 3 of 4 scores it 60.

A wallet's `event_max` is its highest pair score.

## Stage 3: win-rate test

For wallets with at least 5 resolved markets:

```text
p = P(X >= wins | n = resolved_markets, p_null = 0.5)
winrate_score = (1 - p) * 100   if n >= 5 and win_rate >= 0.6, else 0
```

The d4vd wallet won 11 of 16 resolved manipulable markets (p = 0.10), giving `winrate_score = 89.5`. Its `event_max` was 70.

## Final score

```text
score = MAX(event_max, winrate_score)
```

A wallet is flagged at `score >= 60`. Code: `src/detection/score.ts`.

## Why two known insiders are not flagged

Spotify Wrapped (50): of about $5,000 traded on the Weeknd and Drake markets, only $580 landed in markets that pass the strict filter, so E2 drops to 0. Loosening the filter for those market variants, or weighting the win-rate score more, would catch it, at a cost.

DraftKings launch (40): one $8.4k position, won 1 of 1, but the wallet was funded three months before its first manipulable trade. Age (0.6) and total size (0.5) are not enough to clear 60.

Lowering the threshold to 45 would catch 4 of 5 but push the random-wallet false positive rate from 4% to about 15%. The current threshold prefers precision over recall.

## Contracts

Polymarket moved from CTF Exchange V1 to V2 on April 28, 2026 (collateral changed from USDC.e to pUSD). V1 has had no new `OrderFilled` events since. The live indexer subscribes to both:

| Contract | Address |
|---|---|
| V1 regular | `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` |
| V1 neg-risk | `0xC5d563A36AE78145C45a50134d48A1215220f80a` |
| V2 regular | `0xE111180000d2663C0091e4f400237545B87B996B` |
| V2 neg-risk | `0xe2222d279d744050d28e00520010520000310F59` |

V2's `OrderFilled` adds `side`, `tokenId`, `builder` and `metadata`. The live indexer maps V2's (`side`, `tokenId`) back to V1's (`makerAssetId`, `takerAssetId`) so downstream code is unchanged. V2 carries about 140 events per second.

## Pipeline timings (Oct 1 to Nov 30, 2025)

| Stage | Script | Time |
|---|---|---|
| Backfill trades | `trades-backfill-parallel.ts`, 20 workers | about 95 min for 56M events |
| Enrich markets | `enrichMarkets.ts` | about 25 min for 149,554 markets |
| Resolution outcomes | `fillOutcomes.ts` | about 13 min for 72,632 conditions |
| Manipulability | `manipulability.ts` | about 6 s |
| Wallet funding | `funding-backfill.ts` | about 12 min for 12,125 wallets |
| Wallet scores | `score.ts` | about 80 s for 647,389 pairs |
| Validation | `validate.ts` | under 1 s |

About 2.5 hours in total on a laptop. Goldsky GraphQL latency and Alchemy rate limits are the bottleneck.

## Database tables

| Table | Purpose |
|---|---|
| `trades` | Every `OrderFilled` event (maker, taker, token_id, USDC, side, ts) |
| `markets` | Gamma metadata and the winning token |
| `market_manipulability` | 0 to 100 score and `is_manipulable` per market |
| `wallet_first_funding` | First USDC.e transfer received per wallet |
| `wallet_scores` | Composite score and per-signal breakdown |
| `indexer_state` | Resume cursors for backfill and live workers |
