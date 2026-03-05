# Souzu VWAP

Programmable VWAP settlement for DeFi, powered by Chainlink CRE.

A settlement-grade spot exchange where large trades settle at the **12-hour VWAP price** computed by a decentralized Chainlink CRE DON across five major exchanges.

> **Big trades shouldn't carry execution risk — trade at the market's consensus price.**

---

## Demo Video

> [Link to demo video](#) _(3–5 min walkthrough of the CRE workflow simulation and live app)_

**Live App:** [souzu.netlify.app](https://souzu.netlify.app)

**VTN Explorer:** [dashboard.tenderly.co/explorer/vnet/df8e9af3-8def-4a57-a586-ee45398539cd](https://dashboard.tenderly.co/explorer/vnet/df8e9af3-8def-4a57-a586-ee45398539cd)

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
4. **CRE Workflow** (`chainlink-vwap-contract-cre`) fetches 1h OHLCV candles from five CEXes, computes VWAP, applies circuit breakers, and writes a signed price report on-chain via the Forwarder — same workflow code in both modes (see [MockForwarder vs Production CRE](#mockforwarder-vs-production-cre))
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
| [`chainlink-vwap-contract-cre/cmd/server/`](./chainlink-vwap-contract-cre/cmd/server/) | Settler microservice: exposes `POST /settle`, runs `cre workflow simulate`, submits rawReport on-chain |
| [`chainlink-vwap-contract-cre/cmd/trigger/main.go`](./chainlink-vwap-contract-cre/cmd/trigger/main.go) | Production trigger: signs and sends HTTP POST to live CRE DON endpoint |

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

The CRE workflow code (`workflow.go`) is identical in both modes — only the execution environment differs.

**Current (demo):** Backend hourly cron job triggers the settler microservice (`cmd/server`), which runs `cre workflow simulate` on a single node — fetching real CEX data and computing VWAP the same way — then submits the rawReport to `MockKeystoneForwarder`.

**Production (live CRE DON):** CRE native cron job fires an HTTP Trigger across all DON nodes. Each node independently executes `workflow.go`; OCR consensus selects the median → signed report submitted via the real CRE Forwarder with F+1 signature verification.

| | Demo (current) | Production |
|-|---------------|------------|
| Workflow trigger | Backend hourly cron → `cmd/server` | CRE native cron + HTTP Trigger |
| Execution | Single-node `cre workflow simulate` | All DON nodes independently |
| Consensus | None (single result) | OCR — F+1 DON signatures |
| Oracle contract | `ManualVWAPOracle` | `ChainlinkVWAPAdapter` |
| Forwarder | `MockKeystoneForwarder` | CRE Forwarder |

---

### Tenderly VTN Demo

**VTN Explorer:** [dashboard.tenderly.co/explorer/vnet/df8e9af3-8def-4a57-a586-ee45398539cd](https://dashboard.tenderly.co/explorer/vnet/df8e9af3-8def-4a57-a586-ee45398539cd)

Testing a 12-hour settlement window on a real testnet would require waiting 12 hours between each step. The VTN demo solves this with two Tenderly-specific JSON-RPC methods:

- **`evm_setNextBlockTimestamp`** — warps the chain's block timestamp to any point in time instantly
- **`tenderly_setErc20Balance`** — mints test tokens to any address without a faucet

`demo-vtn.sh` uses these to simulate 182 hours of trading activity in seconds, producing all four possible trade states on-chain in a single run. This serves three purposes:

1. **Contract validation** — exercises the full lifecycle (`fill` → oracle price → `settle` / `refund`) in one shot, confirming all contract logic is correct end-to-end
2. **Frontend testing fixture** — gives UI engineers a ready-made set of orders covering every state, so they can test rendering and interactions without manually crafting edge cases
3. **Backend re-index resilience** — by recording the block number just before the first `fill`, the backend indexer can be restarted from that point at any time and reconstruct full state, simulating recovery after downtime

#### What the script does

```
T0+0H    fill A  (startA=T0,    endA=T0+12H)
T0+1H    fill C  (startC=T0+1H, endC=T0+13H)
T0+13H   → warp time → CRE simulate(A) → onReport → settle(A) → fill B
T0+169H  → warp time → fill D
T0+182H  → warp time → CRE simulate(B) → onReport  [freeze here]
```

At T0+182H (frozen state):

| Trade | State | Why |
|-------|-------|-----|
| A | **Settled** | Oracle price set, `settle()` called at T0+13H |
| B | **Ready to Settle** | Oracle price set, `settle()` not called, grace not expired |
| C | **Ready to Refund** | No oracle price, grace period expired 1H ago |
| D | **Pending** | endTime just passed, no oracle price, grace not expired |

Each oracle price is set by calling `simulate-and-forward.sh`, which runs `cre workflow simulate` against real CEX data and submits the rawReport on-chain — the same CRE workflow code used in production.

#### Running the demo

```bash
# Requires DEPLOYER_PRIVATE_KEY, TENDERLY_ADMIN_RPC, SPOT_ADDRESS, MANUAL_ORACLE_ADDRESS in .env
./chainlink-vwap-contract-cre/scripts/demo-vtn.sh
```

#### Configuring frontend and backend to point at VTN

**Frontend `.env`:**

| Key | Value |
|-----|-------|
| `VITE_VWAP_ORACLE_CONTRACT_ADDRESS` | `MANUAL_ORACLE_ADDRESS` |
| `VITE_VWAP_CONTRACT_ADDRESS` | `SPOT_ADDRESS` |
| `VITE_TARGET_CHAIN_ID` | VTN chain ID (from Tenderly dashboard) |
| `VITE_SEPOLIA_RPC_URL` | VTN public RPC URL |

**Backend `.env`:**

| Key | Value |
|-----|-------|
| `APP_CONFIG_ETHEREUM_RPC_URL` | VTN public RPC URL |
| `APP_CONFIG_ETHEREUM_VWAP_RFQ_CONTRACT_ADDR` | `SPOT_ADDRESS` |
| `APP_CONFIG_ETHEREUM_INDEXER_START_BLOCK` | Block number just before the first `fill` tx |

#### Setting oracle prices manually

```bash
# Simulate VWAP and write on-chain for a specific time window
RPC_URL=$TENDERLY_ADMIN_RPC ./chainlink-vwap-contract-cre/scripts/simulate-and-forward.sh "2025-02-27 12:00"
```

End time is floored to the hour; `startTime = endTime − 12h`. Verify:

```bash
cast call $MANUAL_ORACLE_ADDRESS \
  "getPrice(uint256,uint256)(uint256)" \
  $START_TIME $END_TIME \
  --rpc-url $TENDERLY_ADMIN_RPC
```
