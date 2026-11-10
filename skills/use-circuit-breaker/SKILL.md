---
name: use-circuit-breaker
description: Build, test, integrate and deploy the CircuitBreakerHook Uniswap v4 hook from the circuit-breaker repo. Use when you need to work with this hook's contract, wire it into a pool, or deploy it with a mined CREATE2 salt.
---

# CircuitBreakerHook

Halts swapping for a cooldown after the price moves further than a pool is willing to move in one window, and lets liquidity leave the whole time.

Family: Risk. Tags: risk, circuit-breaker, oracle-free, no-admin. License: Apache-2.0. Source: `src/hooks/CircuitBreakerHook.sol`. The machine-readable manifest is `hook.json` (ABI, errors, permissions, properties).

## Build and test

Foundry project (`foundry.toml`, solc 0.8.26, evm `cancun`). From a clone of the repo:

```bash
git clone https://github.com/nirholas/circuit-breaker && cd circuit-breaker
git submodule update --init --recursive   # dependencies live in lib/
forge build
forge test
```

Tests are in `test/CircuitBreakerHook.t.sol`. Use `FOUNDRY_PROFILE=ci forge test` for the larger fuzz run (5000 runs).

## Hook permissions

Uniswap v4 reads a hook's permissions from the low bits of its address. This hook claims: `afterSwap`, `beforeSwap`, `afterInitialize`. It is not upgradeable and has no custom swap data requirement (`hook.json` `properties`).

## Configuring a pool

The hook holds per-pool terms that are fixed for the pool's lifetime. Configure the key first, then initialize the pool:

```solidity
hook.configure(key, CircuitBreakerHook.Config({ /* maxTickMove (uint24), windowSeconds (uint32), cooldownSeconds (uint32) */ }));
poolManager.initialize(key, startingSqrtPriceX96);
```

Signature: `configure((address,address,uint24,int24,address),(uint24,uint32,uint32))`. A pool the hook was never configured for cannot be created (`PoolNotConfigured`). Anyone can configure a key that has no pool yet, so if someone front-runs your terms, pick a different `tickSpacing`, which is a different pool id.

## Deploying

A v4 hook must be deployed at an address whose bits match its permissions, so `script/Deploy.s.sol` mines a CREATE2 salt first. The dry run sends nothing:

```bash
export RPC_URL=https://...        # a chain with a Uniswap v4 PoolManager
forge script script/Deploy.s.sol --rpc-url $RPC_URL
```

Broadcasting spends gas from the signer (`PRIVATE_KEY`, or `--account` / `--ledger`) and is irreversible, so show the user the chain, the predicted hook address and the signer, and get an explicit yes before running:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Re-running is safe: a hook already at its mined address is reported and skipped. If the miner fails with `EvmError: OutOfGas`, add `--gas-limit 30000000000`. `--verify` needs `ETHERSCAN_API_KEY`. The chain must be in the address book `src/libraries/Chains.sol`. `script/Deploy.s.sol` fails with "no Uniswap v4 PoolManager known for this chain" otherwise.

## Identifying a deployed hook

```solidity
IHookMetadata(hookAddress).hookName();
IHookMetadata(hookAddress).specURI();   // points at hook.json
```

## Errors you may meet

`InvalidConfig()`, `InvalidPool()`, `PoolAlreadyInitialized()`, `PoolHalted(uint64)`, `PoolNotConfigured()`.

## Limitation

A halt is a blunt instrument: it stops honest trading as well as the attack, and it leaves the pool arbitrageable the moment it lifts. It is the right trade only where the alternative is a pool drained at a price nobody would have quoted.

More: `docs/integrating.md`, `docs/deploying.md`, and https://circuit-breaker-1bp.pages.dev.
