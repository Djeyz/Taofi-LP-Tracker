# Taofi-LP-Tracket
Purpose & context
single-file HTML application called TaoFi LP Tracker (currently lpTaofi.html) — a read-only dashboard for monitoring Uniswap V3 liquidity positions on the Bittensor EVM network (chain ID 964). The project is an ongoing personal tool with no external collaborators.
Core domain knowledge involved: Uniswap V3 mechanics (tick ranges, liquidity math, impermanent loss), EVM/ethers.js development, Bittensor-specific RPC constraints, and DeFi LP accounting (fees, ROI, APR, capital events).

Key capabilities now in place:

Position tracking: Active and closed positions discovered via TaoStats Blockscout API (NFT enumeration), with on-chain data read via RPC
Event classification: Distinguishes pureCollects (fees only), wcEvts (Collect bundled with DecreaseLiquidity = capital + fees), and withdrawals (capital only) — critical for correct fee/ROI accounting
ROI calculation: Per-position ROI badges in headers, detailed panels for active positions (renderAutoIL), realized ROI banners for closed positions
APR display: Incorporates pending unclaimed fees (feesTao/feesUsdc) for active positions with no collected fees yet
Global stats: Weighted-average IL using on-chain V3 data from positionPerf, weighted by depositedHistUSD; global fees exclude withdrawal-paired Collect events
Historical prices: CryptoCompare integration with ±1 day fallback (histPriceFor)
Favorites system: Named wallets stored in localStorage (taofi_favs), modal for naming/renaming, ⭐ toggle, two-section chip layout (favorites vs. recent) on homepage
Multicall3 batching: Groups positions() calls into single RPC requests with sequential fallback
Architecture: Hybrid RPC (contract reads) + TaoStats Blockscout API (balances, NFT enumeration, event logs)


Tools & resources

ethers.js 5.7.2 (CDN, no wallet connection required)
TaoStats Blockscout API (evm.taostats.io) — primary source for event logs, balances, NFT enumeration
CryptoCompare API — historical TAO price lookup
CoinGecko — secondary price source (EUR conversion, pool-vs-market delta)
Multicall3 at 0xcA11bde05977b3631167028862bE2a173976CA11
Bittensor public RPC nodes: lite.chain.opentensor.ai, entrypoint-finney.opentensor.ai (both unstable; rate-limiting and dropped connections are common)
Output path: /mnt/user-data/outputs/lpTaofi_taostats.html


