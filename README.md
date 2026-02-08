# PrivacySwapHook

Privacy-aware swap execution system on Uniswap v4. Intent-based execution with timing privacy, routing privacy, and liquidity update shielding.

## References

- [Uniswap v4 Overview](https://docs.uniswap.org/contracts/v4/overview) – Hooks, dynamic fees, singleton design, flash accounting
- [v4-template](https://github.com/uniswapfoundation/v4-template) – Template for writing Uniswap v4 Hooks (HookMiner, deployment)
- [OpenZeppelin Uniswap Hooks](https://docs.openzeppelin.com/uniswap-hooks) – BaseHook, fee hooks, sandwich protection
- [Uniswap v4 Security Foundations](https://docs.uniswap.org/contracts/v4/overview) – Hook security, NoOp attacks, delta accounting
- [Uniswap Aggregator Hooks](https://github.com/Uniswap/uniswap-ai) – Hook patterns, protocol integration
- [Unichain Docs](https://docs.unichain.org/docs) – Deployment, routing, bundles, contract addresses

## Features

- **Intent-based swaps**: Execution windows (block ranges), min delay
- **Timing privacy**: Non-deterministic execution timing
- **Routing privacy**: Multiple allowed pools; executor chooses path
- **Liquidity shielding**: LP cooldown before removal
- **MEV-aware**: No delta-return permissions; router allowlisting

## Build

```bash
forge build
forge test
```

## CLI – Real Swap Testing

Run real swaps locally or on testnet:

### Unichain Sepolia (Testnet)

1. **Get testnet tokens**: See [Funding a Wallet](https://docs.unichain.org/docs/getting-started/get-funds-on-unichain). Bridge ETH from Sepolia to Unichain Sepolia, then swap for WETH/USDC on [Uniswap](https://app.uniswap.org).
2. **Run full flow**:
   ```bash
   PRIVATE_KEY=0x... ./cli.sh testnet
   ```
   This deploys the hook, routers, creates a WETH/USDC pool, adds liquidity, and executes both simple and privacy swaps.

### Local (Anvil)

```bash
# Full local flow (Anvil): deploy stack + init pool + liquidity + swap
anvil &                              # Terminal 1
./cli.sh local                       # Terminal 2

# Execute swap against existing deployment
export POOL_MANAGER=0x... SWAP_ROUTER=0x... TOKEN0=0x... TOKEN1=0x... HOOK=0x...
./cli.sh swap
```

| Command | Description |
|---------|-------------|
| `./cli.sh local` | Full flow on local Anvil |
| `./cli.sh testnet` | Full flow on Unichain Sepolia (WETH/USDC, needs bridge first) |
| `./cli.sh testnet-privacy` | Privacy flow: submit intent → deferred execute |
| `./cli.sh deploy-swap` | Deploy fresh contracts + liquidity (run before each swap when you're the only LP) |
| `./cli.sh swap`  | Execute swap (requires env vars from `local`/`testnet` output) |
| `./cli.sh deploy`| Deploy hook only to Unichain Sepolia |

Env vars: `RPC_URL`, `PRIVATE_KEY`, `SWAP_AMOUNT`, `USE_PRIVACY` (true/false for swap).

**Unichain Sepolia Explorer** (chain 1301 – not Ethereum Sepolia):
- [Uniscan](https://sepolia.uniscan.xyz/) | [Blockscout](https://unichain-sepolia.blockscout.com/)

**Broadcast & cache files** (after `./cli.sh testnet`):
- `broadcast/TestnetSwap.s.sol/1301/run-latest.json` – transaction hashes, contract addresses, calldata. Used by Foundry to verify contracts, replay deployments, and for tooling.
- `cache/TestnetSwap.s.sol/1301/run-latest.json` – RPC endpoints and derived metadata. Used for replay/resume. Cache is gitignored.

## Deploy (Unichain)

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for Unichain contract addresses.

```bash
# Unichain Sepolia
forge script script/DeployPrivacySwapHook.s.sol --rpc-url https://sepolia.unichain.org --broadcast
```

## SwapIntent Encoding

```solidity
SwapIntent memory intent = SwapIntent({
    startBlock: uint64(block.number + 5),
    endBlock: uint64(block.number + 20),
    minDelayBlocks: 2,
    createdAtBlock: 0,
    allowedPoolIds: [poolId1, poolId2, poolId3, poolId4],
    allowedPoolCount: 2,
    minAmountOut: 1e18,
    salt: bytes32(0)
});
bytes memory hookData = abi.encode(intent);
```

## Unichain eth_sendBundle

The execution window aligns with [Unichain's eth_sendBundle](https://docs.unichain.org/docs/technical-information/advanced-txn):

```javascript
const bundleParams = {
  txs: [signedSwapTx],
  minBlockNumber: `0x${(currentBlock + 5).toString(16)}`,
  maxBlockNumber: `0x${(currentBlock + 20).toString(16)}`
};
// Revert protection: no gas if swap reverts
const txHash = await provider.send('eth_sendBundle', [bundleParams]);
```

## License

MIT
