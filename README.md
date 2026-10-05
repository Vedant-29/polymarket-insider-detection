# Polymarket Insider Detection

A TypeScript and Postgres pipeline that indexes every `OrderFilled` trade on Polymarket, finds markets one person could influence, and scores every wallet for insider-like trading. It runs on historical data and live, printing a `[FLAG]` line when a new trade crosses the threshold.

## Results

On October 1 to November 30, 2025 (56 million trades, 619,834 wallets scored), it flags 3 of the 5 known insider wallets that traded in that window at a threshold of 60. The other 2 still rank in the top 1% of all wallets.

| Wallet | Score | Flagged | Win rate |
|---|---|---|---|
| d4vd Google Year in Search | 93.3 | yes | 11 / 16 |
| MicroStrategy BTC sale | 86.7 | yes | 1 / 1 |
| Trump pardon CZ | 60.0 | yes | 1 / 2 |
| Spotify Wrapped #3 | 50.0 | no | 2 / 2 |
| DraftKings launch | 40.0 | no | 1 / 1 |

A random control sample of 100 wallets (3+ trades) has a mean of 23, a 95th percentile of 56, and 4 flagged. Full output is in [docs/validation-results.json](docs/validation-results.json).

## How it works

- Index trades from Polymarket's V1 and V2 exchanges: historical via the Goldsky orderbook subgraph, live via an Alchemy WebSocket on Polygon.
- Look up each wallet's first USDC.e deposit (Alchemy `getAssetTransfers`) to measure wallet age.
- Enrich markets from the Polymarket Gamma API (question, end date, liquidity, outcome).
- Keep only manipulable markets, then score each (wallet, market) on wallet age, trade size, entry timing and total stake, plus a binomial win-rate test across markets.

Details, the reasoning behind the misses, and contract addresses are in [docs/algorithm.md](docs/algorithm.md).

## Requirements

- Node 20+
- Postgres 16 (`brew install postgresql@16` on macOS)
- An Alchemy API key with Polygon enabled (free tier works)

## Setup

```sh
git clone https://github.com/Vedant-29/polymarket-insider-detection.git
cd polymarket-insider-detection
cp .env.example .env        # set ALCHEMY_API_KEY and DATABASE_URL
createdb polymarket_insider
npm install
npm run db:migrate
```

`db:migrate` creates the six tables (`trades`, `markets`, `market_manipulability`, `wallet_first_funding`, `wallet_scores`, `indexer_state`).

## Environment variables

| Variable | Required | What it is for | Where to get it |
|---|---|---|---|
| `ALCHEMY_API_KEY` | Yes | Polygon RPC and WebSocket, funding lookups | alchemy.com > Create app > Polygon Mainnet |
| `DATABASE_URL` | Yes | Postgres connection | Default `postgres://localhost:5432/polymarket_insider` |
| `BACKFILL_MONTHS` | No | Months back for the single-cursor backfill (default 9) | Choose a value |
| `FUNDING_MIN_USDC` | No | Minimum biggest trade for a wallet to get a funding lookup (default 5000) | Choose a value |

## Usage

Run the full historical pipeline. These are the steps behind the results above (about 2.5 hours on a laptop).

```sh
npm run backfill:trades -- --workers 20 --from 2025-10-01T00:00:00Z --to 2025-11-30T23:59:59Z
psql -d polymarket_insider -c "ANALYZE trades;"
npm run enrich:markets
npm run enrich:outcomes
npm run score:manipulability
npm run backfill:funding
npm run score:wallets
npm run validate
npm run leaderboard
```

Run `ANALYZE` after the backfill. The later queries are slow without it.

If a backfill worker dies on a transient `fetch failed`, resume only that worker:

```sh
npm run backfill:trades:resume -- --name trades-w4 --to 2025-10-16T05:59:59Z
```

Watch trades live and flag them as they happen:

```sh
npm run live
```

```text
[live] subscribing to OrderFilled on V1 + V2 (regular + neg-risk)…
[live v2] 2026-05-09T14:43:54.000Z BUY $939.06 for 999.04 shares @ 0.94 (token 14099837…)
[FLAG] wallet=0x44ab68a9… score=60.0 e1_age=0.00 e2_size=0.00 e3_timing=1.00 e4_total=0.00 marketUsd=523 q=…
```

## Notes

- The indexed window is October to November 2025. Three of the eight wallets in `data/known_insiders.json` (Maduro 1, Maduro 2, Israel/Iran reactivation) traded in January 2026 and are outside it.
- Spotify Wrapped and DraftKings are missed on purpose at the current threshold. Lowering it to 45 catches 4 of 5 but raises the random false positive rate from 4% to about 15%.
- Since April 28, 2026 Polymarket trades only on the V2 exchange. The live indexer handles both event shapes.

## License

No license file yet.
