# CircuitBreaker

**Halts swapping for a cooldown after the price moves further than a pool is willing to move in one window, and lets liquidity leave the whole time.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://circuit-breaker-1bp.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/CircuitBreakerHook.sol`](src/hooks/CircuitBreakerHook.sol)
- **Licence:** MIT

## How it works

Every venue outside crypto stops trading after a limit move, for a reason that has nothing to do with paternalism: a violent move is usually either an error or an attack, and the cheapest defence against both is to stop, let information arrive, and start again. On-chain the same event is normally handled by a governance multisig that pauses a contract minutes after it mattered. This hook makes the rule mechanical and local to one pool.

It keeps a reference tick, refreshed at most once per `windowSeconds`. After every swap it compares the new tick to that reference. If the pool moved further than `maxTickMove`, swapping halts for `cooldownSeconds` and then resumes on its own.

There is no admin, no pause key and no way for anyone, including the deployer, to halt a pool that has not moved or to extend a halt that has expired. The design decision worth stating: the swap that breaches the limit is allowed to complete. Reverting it instead would turn the hook into a price cap, and a price cap on an AMM is a strictly worse instrument than a halt.

It cannot be enforced (the same move arrives as several smaller swaps), it strands the pool at a price the market has left, and it guarantees that the arbitrage against the pool stays open and profitable for as long as the cap holds. Halting after the fact gives up the last swap and buys the thing that actually matters, which is time. Liquidity operations are never blocked.

A provider can withdraw during a halt, which is the property that makes this safe to use: the worst case for someone caught in a halted pool is that they exit rather than trade. 0001^1`), so `maxTickMove = 500` is a five percent move. Prior art: pause-guardian patterns are everywhere and oracle-deviation checks exist as hooks.

An autonomous, self-clearing, per-pool halt with no privileged role and no oracle does not.

## Prior art

Pause-guardian patterns are everywhere and oracle-deviation checks exist as hooks. An autonomous, self-clearing, per-pool halt with no privileged role and no oracle does not.

## Where it does not help

A halt is a blunt instrument: it stops honest trading as well as the attack, and it leaves the pool arbitrageable the moment it lifts. It is the right trade only where the alternative is a pool drained at a price nobody would have quoted.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    CircuitBreakerHook.Config({
        maxTickMove: /* uint24 */ 0,
        windowSeconds: /* uint32 */ 0,
        cooldownSeconds: /* uint32 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `maxTickMove` | `uint24` | ticks |
| `windowSeconds` | `uint32` | seconds |
| `cooldownSeconds` | `uint32` | seconds |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `InvalidConfig()` | `maxTickMove`, `windowSeconds` and `cooldownSeconds` must all be non-zero. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolHalted(uint64)` | Swapping is halted until `until`. Liquidity may still be added or removed. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `beforeSwap`
- `afterSwap`

Mask: `0x10c0`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # CircuitBreaker
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # risk, circuit-breaker, oracle-free, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/circuit-breaker
cd circuit-breaker
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
