# Foundry contracts

Two example contracts for Elysium testnet. Neither has an owner, a mint role, a pause, or an upgrade path.

| Contract | Role |
| --- | --- |
| `src/ElysiumStreamingToken.sol` | ERC-20 with permissionless time-based accrual after a fixed initial supply. |
| `src/ElysiumQuoteBook.sol` | Two-sided quote book with a minimum refresh interval. Quotes are whatever a caller publishes. |

```bash
forge test
forge build
./deploy.sh              # needs ELYSIUM_TESTNET_KEY in the repo .env
```

Addresses and receipts: `deployed.json`. Parent README: `../README.md`.
