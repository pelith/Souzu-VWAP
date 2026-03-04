# Chainlink VWAP RFQ Spot

> **Big trades shouldn't carry execution risk — trade at the market's consensus price.**

A settlement-grade spot exchange where large trades settle at the **12-hour VWAP price** computed by a decentralized Chainlink CRE DON across five major exchanges.

---

## Demo Video

> [Link to demo video](#) _(3–5 min walkthrough of the CRE workflow simulation and live app)_

**Live App:** [souzu.netlify.app](https://souzu.netlify.app)

---

## Why VWAP?

According to _The TRADE Algorithmic Trading Survey_, over **70% of traders** use VWAP as their primary execution algorithm for low-urgency trades — it is the industry standard for large block settlement.

On-chain, large trades face a fundamental problem: any real-time price feed exposes the taker to execution risk. VWAP removes this by settling at the market's volume-weighted consensus price over a 12-hour window, eliminating price impact on the settlement itself.

- **Taker:** When the trade is big, market impact gets expensive — VWAP gives you the market's consensus price.
- **Maker:** When the trade is big, execution matters — beat the market's consensus and keep the edge.

---

## Why CRE?

**VWAP must see the whole market, not just the chain.**
Real volume lives on Binance, Coinbase, and OKX — not on DEXes. On-chain DEX VWAP is not only incomplete, it can be manipulated with a flash loan wash trade at near-zero cost.

**VWAP is not a price an oracle can continuously publish.**
Traditional oracles push a single reference price. VWAP changes with every trade's time window — there is no single "the VWAP" that can be pre-computed and cached. It must be calculated on demand, per settlement.

**Only CRE makes this possible in a decentralized way.**
Each CRE node independently fetches historical candle data from five CEX APIs and computes the VWAP — no shared signer, no central aggregator. OCR consensus ensures no single node can influence the result. Without CRE, VWAP is just one server's word. With CRE, it becomes a price confirmed by the entire network.

---

## How It Works

1. **Maker** signs an EIP-712 order off-chain (amount, min out, delta bps, deadline) and submits it to the orderbook
2. **Taker** calls `fill()` — contract verifies the signature, locks WETH + USDC, records `startTime` / `endTime` (+12h)
3. **Backend** detects the `Filled` event; an **hourly cron job** scans for expired orders and triggers the settler
4. **CRE Workflow** executes on each DON node independently:
   - HTTP Trigger fires on-demand — no continuous gas cost
   - Each node independently fetches 1h OHLCV candles from Binance, OKX, Bybit, Coinbase, Bitget
   - Each node independently computes VWAP = Σ(quoteVol) / Σ(baseVol)
   - Four circuit breakers applied: coverage gate (≥3 venues), outlier scrubbing (>2% deviation), staleness check (≤60min from endTime), flash crash guard (<15% VWAP vs last close)
   - OCR consensus → median selected → signed report
5. **Forwarder** writes the signed report on-chain via `onReport()`
6. **Anyone** calls `settle()` — `VWAPRFQSpot` reads oracle price, applies `deltaBps`, distributes funds to maker and taker
7. If no one settles within the grace period, **anyone** calls `refund()` — both parties reclaim their original deposits

---

## Architecture

```mermaid
flowchart TB
    Maker(["Maker"]) -- "Sign order via<br>EIP-712<br>off-chain" --> OB["Backend: Orderbook<br>"]
    OB -- Maker cancels --> Cancel["Maker calls<br>cancelOrderHash<br>Order marked as used, cannot be<br>filled"]
    OB -- Taker accepts order --> Taker(["Taker"])
    Taker -- Fill order, signature,<br>takerAmountIn --> Contract["VWAPRFQSpot Contract<br>Verify signature<br>Lock maker + taker funds<br>Emit Filled event"]
    Contract --> Cron["Backend: <br>hourly cron job<br>Call ChainLink CRE to settle VWAP price"]
    Cron -- POST /settle --> Settler["CRE Settler"]
    Settler --> Binance["Binance"] & OKX["OKX"] & Bybit["Bybit"] & Coinbase["Coinbase"] & Bitget["Bitget"]
    Binance --> Report["VWAP Price calculation:<br>Filter outliers → Median<br>Build rawReport<br>MockKeystoneForwarder.report"]
    OKX --> Report
    Bybit --> Report
    Coinbase --> Report
    Bitget --> Report
    Report --> OnChain["Forwarder:<br>Call VWAPOracle.onReport<br>to stores price"]
    OnChain -- <br> --> Settle@{ label: "Anyone calls<br>VWAPRFQSpot.settle" }
    Settle -- No one settles<br>Grace period<br>expired --> Refund("Maker / Taker<br>reclaim original deposits")
    Settle --> n1@{ label: "Calculate settlement price with base point." }
    n1 -- Success --> Payout("Maker / Taker<br>receive funds")

    Settle@{ shape: diamond}
    n1@{ shape: rect}
```

---

## Chainlink Integration

### CRE Workflow

| File | Description |
|------|-------------|
| [`chainlink-vwap-contract-cre/vwap-eth-quote-flow/workflow.go`](./chainlink-vwap-contract-cre/vwap-eth-quote-flow/workflow.go) | Main CRE Workflow: fetches multi-exchange OHLCV, computes 12h VWAP, applies circuit breakers, writes on-chain via OCR |
| [`chainlink-vwap-contract-cre/vwap-eth-quote-flow/workflow.yaml`](./chainlink-vwap-contract-cre/vwap-eth-quote-flow/workflow.yaml) | CRE CLI workflow config (targets, trigger settings) |
| [`chainlink-vwap-contract-cre/project.yaml`](./chainlink-vwap-contract-cre/project.yaml) | CRE CLI project settings (RPC, forwarder addresses for Sepolia) |
| [`chainlink-vwap-contract-cre/cmd/trigger/main.go`](./chainlink-vwap-contract-cre/cmd/trigger/main.go) | Backend trigger: signs and sends HTTP POST to CRE DON endpoint |

### Smart Contracts

| File | Description |
|------|-------------|
| [`chainlink-vwap-contract-cre/contracts/evm/src/ChainlinkVWAPAdapter.sol`](./chainlink-vwap-contract-cre/contracts/evm/src/ChainlinkVWAPAdapter.sol) | Production oracle — receives signed VWAP reports from CRE Forwarder via `IReceiver` |
| [`chainlink-vwap-contract-cre/contracts/evm/src/keystone/IReceiver.sol`](./chainlink-vwap-contract-cre/contracts/evm/src/keystone/IReceiver.sol) | Chainlink Keystone `IReceiver` interface |
| [`chainlink-vwap-contract-cre/contracts/evm/src/VWAPRFQSpot.sol`](./chainlink-vwap-contract-cre/contracts/evm/src/VWAPRFQSpot.sol) | Main exchange contract — reads oracle price on `settle()` |
| [`chainlink-vwap-contract-cre/contracts/evm/src/ManualVWAPOracle.sol`](./chainlink-vwap-contract-cre/contracts/evm/src/ManualVWAPOracle.sol) | Staging oracle — implements `IReceiver`, used for simulation and demo |

---

## Deployed Contracts (Sepolia)

| Contract | Address |
|----------|---------|
| ManualVWAPOracle | [`0xd7D42352bB9F84c383318044820FE99DC6D60378`](https://sepolia.etherscan.io/address/0xd7D42352bB9F84c383318044820FE99DC6D60378) |
| VWAPRFQSpot | [`0x61A73573A14898E7031504555c841ea11E7FB07F`](https://sepolia.etherscan.io/address/0x61A73573A14898E7031504555c841ea11E7FB07F) |
| MockKeystoneForwarder (Chainlink) | [`0x15fC6ae953E024d975e77382eEeC56A9101f9F88`](https://sepolia.etherscan.io/address/0x15fC6ae953E024d975e77382eEeC56A9101f9F88) |

---

## Quick Start

```bash
git clone --recurse-submodules https://github.com/pelith/Souzu-VWAP
```

If you already cloned without `--recurse-submodules`, run:

```bash
git submodule update --init --recursive
```

| Component | Setup Guide |
|-----------|-------------|
| CRE Workflow & Contracts | [chainlink-vwap-cre/README.md](./chainlink-vwap-cre/README.md) |
| Backend | [chainlink-vwap-be/README.md](./chainlink-vwap-be/README.md) |
| Frontend | [chainlink-vwap-fe/README.md](./chainlink-vwap-fe/README.md) |

---

## Appendix

### CRE Workflow Simulation

Run a live simulation without a CRE account — fetches real CEX data and computes current VWAP:

```bash
cd chainlink-vwap-contract-cre

# Unit tests
cd vwap-eth-quote-flow && go test -v

# CLI simulation (past 12h, real exchange data)
cre workflow simulate vwap-eth-quote-flow \
  --non-interactive --trigger-index 0 \
  --http-payload '{"startTime":1739552400,"endTime":1739595600}' \
  --target staging-settings
```

### Scripts

| Script | Description |
|--------|-------------|
| [`scripts/simulate.sh`](./chainlink-vwap-contract-cre/scripts/simulate.sh) | Run CRE simulate with a custom payload |
| [`scripts/simulate-and-forward.sh`](./chainlink-vwap-contract-cre/scripts/simulate-and-forward.sh) | Simulate + submit rawReport on-chain via MockKeystoneForwarder |
| [`scripts/demo-vtn.sh`](./chainlink-vwap-contract-cre/scripts/demo-vtn.sh) | Create demo orders on Tenderly VTN in four settlement states |

### MockForwarder vs Production CRE

The current Sepolia deployment routes reports through `MockKeystoneForwarder`, which accepts unsigned reports and calls `onReport()` directly — useful for simulation and demos without a live CRE DON.

In production, `ChainlinkVWAPAdapter` replaces `ManualVWAPOracle` as the receiver. The real CRE Forwarder verifies F+1 DON signatures before calling `onReport()`, providing full decentralized trust guarantees.

| | Demo (current) | Production |
|-|---------------|------------|
| Oracle contract | `ManualVWAPOracle` | `ChainlinkVWAPAdapter` |
| Forwarder | `MockKeystoneForwarder` | CRE Forwarder |
| Signature verification | None | F+1 DON signatures |
| Report source | `simulate-and-forward.sh` | Live CRE DON nodes |
