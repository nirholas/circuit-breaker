# CircuitBreaker

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/circuit-breaker?style=flat&logo=github)](https://github.com/nirholas/circuit-breaker/stargazers) [![License](https://img.shields.io/github/license/nirholas/circuit-breaker?style=flat)](https://github.com/nirholas/circuit-breaker/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/circuit-breaker?style=flat)](https://github.com/nirholas/circuit-breaker/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/circuit-breaker/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/circuit-breaker/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Halts swapping for a cooldown after the price moves further than a pool is willing to move in one window, and lets liquidity leave the whole time.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://circuit-breaker-1bp.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/CircuitBreakerHook.sol`](src/hooks/CircuitBreakerHook.sol)
- **Licence:** Apache-2.0

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

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/circuit-breaker&type=Date)](https://www.star-history.com/#nirholas/circuit-breaker&Date)

<!-- three.ws:growth -->
## Support the project

If circuit-breaker saves you time, **[star it on GitHub](https://github.com/nirholas/circuit-breaker)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=circuit-breaker%3A%20Halts%20swapping%20for%20a%20cooldown%20after%20the%20price%20moves%20further%20than%20a%20pool%20is%20willing%20to%20move%20in%20one&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcircuit-breaker) · [Share on Bluesky](https://bsky.app/intent/compose?text=circuit-breaker%3A%20Halts%20swapping%20for%20a%20cooldown%20after%20the%20price%20moves%20further%20than%20a%20pool%20is%20willing%20to%20move%20in%20one%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fcircuit-breaker) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcircuit-breaker) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcircuit-breaker&t=circuit-breaker%3A%20Halts%20swapping%20for%20a%20cooldown%20after%20the%20price%20moves%20further%20than%20a%20pool%20is%20willing%20to%20move%20in%20one) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcircuit-breaker&title=circuit-breaker%3A%20Halts%20swapping%20for%20a%20cooldown%20after%20the%20price%20moves%20further%20than%20a%20pool%20is%20willing%20to%20move%20in%20one)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/circuit-breaker` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/circuit-breaker/issues) or [start a discussion](https://github.com/nirholas/circuit-breaker/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/circuit-breaker)](https://github.com/nirholas/circuit-breaker/graphs/contributors)

<!-- /three.ws:growth -->
