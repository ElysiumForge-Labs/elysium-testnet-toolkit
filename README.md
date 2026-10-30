# elysium-testnet-toolkit

Probes, measurements, and example contracts for Kinetiq's Elysium testnet — an Arbitrum Orbit L2 that settles to HyperEVM.

Part of [ElysiumForge Labs](https://github.com/ElysiumForge-Labs).

> **Independent research toolkit.** Not affiliated with, endorsed by, or sponsored by Kinetiq or Hyperliquid. Figures below are testnet measurements. They do not describe mainnet performance.

The scripts report what the chain does today, not what the docs claim it will do.

## Chain

Verified against the public RPC on 2026-09-26.

| | |
| --- | --- |
| Chain ID | `99801` |
| Public RPC | `https://testnet-rpc.elysium.kinetiq.xyz` |
| Explorer | `https://elysium.kinetiq.xyz/testnet-explorer` |
| Faucet | `https://elysium.kinetiq.xyz/testnet-faucet` |
| Native gas | HYPE |
| Stack | Arbitrum Orbit (Nitro / ArbOS), sequencer operated by Conduit |
| Client | `nitro/vdevelopment/linux-amd64/go1.25.14` |
| Parent chain | HyperEVM testnet, chain ID `998` |
| Measured block time | about 420–600 ms (docs claim 100–200 ms) |
| baseFee | 0.01 gwei, flat |

Two Conduit endpoints are live and are not listed in Kinetiq's docs. Both answer without a key. A key from [Conduit Nodes](https://app.conduit.xyz/nodes) lifts rate limits (free tier: 100M compute units per month).

| | |
| --- | --- |
| JSON-RPC | `https://rpc-elysium-testnet.t.conduit.xyz` |
| Sequencer feed | `wss://relay-elysium-testnet.t.conduit.xyz/` |

The feed is not JSON-RPC. The wire format is `{version, messages[]}`. Messages arrive on connect. Do not send RPC methods to it.

Override endpoints with `ELYSIUM_RPC` and `ELYSIUM_FEED`. Bun loads `.env` automatically.

## Not live on testnet

These surfaces ship in an ArbOS upgrade roughly four weeks after mainnet. Nothing here can call them yet.

- HyperCore market-data precompile (order books, prices, balances at HyperCore granularity)
- `ElysiumCoreWriter` predeploy (on-chain HyperCore order placement with callbacks)

## Requirements

- [Bun](https://bun.sh)
- [Foundry](https://book.getfoundry.sh/getting-started/installation) for contracts
- Optional: a Conduit API key and a 0xArchive API key

```bash
bun install
cp .env.example .env
./scripts/install-hooks.sh
```

`.env` is gitignored. Never commit it.

| Variable | Purpose |
| --- | --- |
| `CONDUIT_API_KEY` | Optional. Appended to the Conduit RPC and feed URLs. |
| `ELYSIUM_TESTNET_KEY` | Throwaway testnet key used by `bun run wallet` and `foundry/deploy.sh`. |
| `OXARCHIVE_API_KEY` | Optional. HyperCore data via 0xArchive (`OX_ARCHIVE_API_KEY` is also accepted). |
| `ELYSIUM_RPC` / `ELYSIUM_FEED` | Optional endpoint overrides. |

## Commands

```bash
bun run probe            # chain health and documented contract code
bun run endpoints        # Kinetiq vs Conduit RPC latency
bun run blocktime        # block time vs the 100–200 ms claim
bun run log              # append one row to data/blocktime.csv
bun run log:summary      # trend the committed series
bun run feed             # consume the Conduit sequencer feed
bun run bridge           # bridge route math on both chains (read-only)
bun run wallet           # generate or load a testnet key; report balances
bun run economics        # cost per transaction shape
bun run universe         # assets present on both Elysium and HyperCore
bun run arb:feasibility  # Elysium-leg delay vs HyperCore price movement
```

`bun run wallet --new` replaces the key in `.env`. The private key is never printed. Import it into a dedicated browser wallet only; the faucet is a wallet-connect page, not an address form. Use the public RPC in the wallet, not the keyed Conduit URL.

## Secret handling

`.env` is gitignored. A pre-commit hook at `scripts/pre-commit-hook` blocks secrets from entering git history:

1. Exact match against values in `.env` (skips public URLs).
2. gitleaks, if installed, for generic key patterns.
3. Refusal to commit `.env`, `*.key`, or `*.pem`.

Install after a clone with `./scripts/install-hooks.sh`. Layer 1 works without gitleaks. Do not widen `.gitleaks.toml` to make a scan pass.

## Documented bridge addresses

On Elysium (99801):

| Name | Address |
| --- | --- |
| L2GatewayRouter | `0x89659883a9d980925733B0A698F117AAb65ac718` |
| ElysiumBridgeFactory | `0xb94A38a4aC46970559E89E566f2486a3Fc56BE5a` |

On HyperEVM testnet (998). Probing these against the Elysium RPC correctly returns no code:

| Name | Address |
| --- | --- |
| L1GatewayRouter | `0x1aAE2caD8B0249905492087EF230FcCEa3707C45` |
| Standard gateway | `0x792487eF5E423104bec94e8c58E814e5E0450cEA` |
| ElysiumMirrorFactory | `0xcaDb9986F3727177d48FA07294E1730f9D19290b` |

HyperCore deposit factory addresses are not published.

## Contracts

`foundry/` contains two contracts with no owner, mint role, pause, or upgrade path.

| Contract | Role |
| --- | --- |
| `ElysiumStreamingToken` (OEX) | Fixed initial supply, then permissionless time-based accrual. |
| `ElysiumQuoteBook` | Caller-published two-sided quotes with a minimum refresh interval and staleness. Not an oracle. |

```bash
cd foundry
forge test
forge build
./deploy.sh              # broadcasts; reads ELYSIUM_TESTNET_KEY from the repo .env
```

Deployed 2026-09-27. Addresses and receipts are in `foundry/deployed.json`.

| Contract | Address | Code |
| --- | --- | --- |
| ElysiumStreamingToken | `0xdc5a850c1cce4e7fe6a35976763698cd554864be` | 2,821 B |
| ElysiumQuoteBook | `0xb5681268c210e14732e7c1e0a0847eef764dbc5e` | 1,898 B |

The testnet explorer has no source-verification API. `forge verify-contract` cannot succeed. This repository and `deployed.json` are the record of what was deployed.

## Measurements

`data/blocktime.csv` is a committed time series. `bun run log` appends one 200-block window. `blockTimeMs` is an aggregate over that window, not a per-block gap. Nitro often stamps several consecutive blocks with the same second, so a per-block distribution is mostly zeros.

`bun run arb:feasibility` compares HyperCore price movement inside one Elysium block (~440 ms) to an assumed AMM spread. With a 30 bp spread, measured tails do not clear the required dislocation. The live spread is not measurable until an AMM exists; that input can reverse the conclusion. The measurement tool is the deliverable. Do not treat this as a reason to ship an arb executor against testnet.

Further notes: `docs/findings.md`, `docs/staking-economics.md`.

## Layout

```
src/                 probes and measurements (Bun / TypeScript)
scripts/             pre-commit hook
data/blocktime.csv   committed block-time series
foundry/             Solidity, tests, deploy.sh, deployed.json
docs/                research notes
```

## Open questions

1. Will block time reach the documented 100–200 ms before mainnet?
2. When does the ArbOS upgrade ship, and when are precompile addresses published?
3. Do mainnet addresses and testnet state carry over?

## License

Independent research under ElysiumForge Labs. Treat measurements as observations, not as a product or as investment advice.
