# TaoFi LP Tracker

A self-contained, read-only dashboard for monitoring **Uniswap V3 liquidity positions** on the **Bittensor EVM network** (chain ID 964), specifically for the **TAO/USDC pool** on [TaoFi](https://astrid.taofi.com/pool).

No wallet connection required — just enter any EVM address to inspect its LP positions.

This dashboard was created with Claude sonnet 4.6 and almost entierly vite code.

---

## Features

- **Position discovery** — automatically finds all active and closed LP positions for any EVM wallet via the TaoStats Blockscout API (NFT Transfer event scanning)
- **Live position data** — current liquidity, token amounts, price range, in-range/out-of-range status
- **Unclaimed fees** — real-time pending fees (WTAO + USDC) from on-chain `positions()` calls
- **Fee history** — full collect event history per position, with event classification:
  - `pureCollect` — fee-only harvests
  - `wcEvt` — collect bundled with a DecreaseLiquidity (capital + fees)
  - `withdrawal` — capital-only removal
- **ROI per position** — realized return on invested capital, accounting for capital events correctly (no double-counting)
- **APR** — annualized yield including pending unclaimed fees for active positions
- **Impermanent loss** — calculated from real on-chain token amounts (Uniswap V3 formula), weighted-average IL displayed globally
- **Historical prices** — TAO/USD price at deposit date via CryptoCompare (with ±1 day fallback)
- **Global stats** — portfolio-level summary: total value, total fees collected, weighted IL, active vs closed counts
- **Favorites system** — name and save wallet addresses locally (stored in browser `localStorage`)
- **Recent addresses** — quick-access history of recently searched wallets

---

## Quick Start

1. Download `lpTaofi.html`
2. Open it in any modern browser (Chrome, Firefox, Brave, Edge)
3. Enter an EVM wallet address (`0x…`) in the search bar
4. Optionally save it as a favorite with a custom name ⭐

No installation, no build step, no server required.

> ⚠️ **Substrate / SS58 addresses are not supported.** TaoFi uses EVM H160 addresses (`0x…`). You can convert your Substrate address on [evm.taostats.io](https://evm.taostats.io) → *Address Converter*, or use your MetaMask / Rabby address directly.

---

## Architecture

This is a **single-file HTML application** with zero build dependencies. Everything is inlined.

| Layer | Technology |
|---|---|
| Web3 / contract reads | ethers.js 5.7.2 (CDN, UMD build) |
| Event logs / NFT enumeration | TaoStats Blockscout API (`evm.taostats.io/api`) |
| Historical TAO price | CryptoCompare `histoday` API (public, no key required) |
| Current TAO price | CoinGecko public API |
| Batch RPC calls | Multicall3 (`0xcA11bde05977b3631167028862bE2a173976CA11`) |
| State persistence | Browser `localStorage` (favorites, config, search history) |

### Data flow

```
User enters wallet address
        │
        ▼
TaoStats API ──► NFT Transfer events ──► Token IDs owned
        │
        ▼
RPC (sequential) ──► positions() via Multicall3 ──► Liquidity, ticks, fees
        │
        ▼
TaoStats API ──► Collect / IncreaseLiquidity / DecreaseLiquidity logs
        │
        ▼
CryptoCompare ──► Historical TAO price at deposit date
        │
        ▼
On-chain math ──► IL, ROI, APR, fee totals
```

---

## Contract Addresses (Bittensor EVM, chain 964)

| Contract | Address |
|---|---|
| TaoFi NftManager (Uniswap V3 Position Manager) | `0x61EeA4770d7E15e7036f8632f4bcB33AF1Af1e25` |
| TAO/USDC Pool | `0x6647dcbeb030dc8E227D8B1A2Cb6A49F3C887E3c` |
| WTAO (Wrapped TAO) | `0x9Dc08C6e2BF0F1eeD1E00670f80Df39145529F81` |
| USDC | `0xB833E8137FEDf80de7E908dc6fea43a029142F20` |
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` |

**Token ordering:** token0 = WTAO (18 decimals), token1 = USDC (6 decimals)  
WTAO is lexicographically smaller than USDC, so it occupies the token0 slot — the opposite of what one might assume.

---

## Known Limitations

- **`eth_getLogs` is unavailable** on Bittensor public RPC nodes — all log fetching goes through the TaoStats Blockscout API
- **RPC instability** — Bittensor public nodes (`lite.chain.opentensor.ai`, `entrypoint-finney.opentensor.ai`) are prone to rate-limiting and dropped connections; all calls are made strictly sequentially with automatic retry and failover
- **TaoFi NftManager does not implement ERC721Enumerable** — NFT discovery relies on Transfer event scanning, not `tokenOfOwnerByIndex`
- **Read-only** — this tool cannot submit transactions or connect a wallet
- **Single pool** — currently scoped to the TAO/USDC pool only

---

## RPC Endpoints Used

```
https://lite.chain.opentensor.ai
https://entrypoint-finney.opentensor.ai
```

Both are public Bittensor EVM nodes. No API key required.

---

## External APIs Used

| Service | Endpoint | Auth |
|---|---|---|
| TaoStats Blockscout | `https://evm.taostats.io/api` | None |
| CryptoCompare | `https://min-api.cryptocompare.com/data/v2/histoday` | None (public) |
| CoinGecko | `https://api.coingecko.com/api/v3/simple/price` | None (public) |

All APIs are used on their free, unauthenticated tiers. No API keys are stored in this app.

---

## Privacy

All data stays in your browser:
- No telemetry, no analytics, no external data collection
- Favorites and search history are stored in your browser's `localStorage` only
- Wallet addresses you search are sent only to the TaoStats API and the Bittensor RPC nodes listed above (necessary to fetch on-chain data)

---

## Development

The entire application is `lpTaofi.html`. Open it in a browser to run it. No build tools or local server needed.

To modify:
1. Edit the HTML file in any text editor
2. Reload the browser
3. Check the browser console — a built-in `dbg()` logger outputs tagged `[TAOFI]` messages for all RPC calls, retries, and data processing steps

### Key implementation notes for contributors

- **All RPC calls must be sequential** — `Promise.all` causes interference on Bittensor nodes
- **Create ethers contract instances fresh per call** (inline arrow functions) — stale provider bindings after reconnection cause silent failures
- **Two distinct RPC error types**: `query timeout exceeded` (chunk too large → retry same range) vs. `missing response` (RPC saturation → switch endpoint + wait)
- **`positionPerf[String(tid)]`** is the central per-position cache object — authoritative source for computed data (ROI, IL, APR, fee totals)
- **Event classification matters** — `DecreaseLiquidity` capital amounts must not be added alongside the `Collect` events that already include that capital (double-counting pitfall)

---

## License

MIT — see [LICENSE](LICENSE)

---

## Acknowledgements

Built on top of:
- [TaoFi](https://taofi.xyz) — Uniswap V3 fork on Bittensor EVM
- [TaoStats](https://taostats.io) — Bittensor explorer and API
- [ethers.js](https://docs.ethers.org/v5/) — Ethereum JavaScript library
- [Uniswap V3](https://uniswap.org) — concentrated liquidity AMM
