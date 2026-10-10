# CommitRevealDark

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/commit-reveal-dark?style=flat&logo=github)](https://github.com/nirholas/commit-reveal-dark/stargazers) [![License](https://img.shields.io/github/license/nirholas/commit-reveal-dark?style=flat)](https://github.com/nirholas/commit-reveal-dark/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/commit-reveal-dark?style=flat)](https://github.com/nirholas/commit-reveal-dark/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/commit-reveal-dark/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/commit-reveal-dark/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Every swap must have been committed to in an earlier block, so nobody can react to an order they can see.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://commit-reveal-dark.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/CommitRevealDarkHook.sol`](src/hooks/CommitRevealDarkHook.sol)
- **Licence:** Apache-2.0

## How it works

Sandwiching works because of a timing asymmetry, not because of secrecy. Your order is visible before it executes, and an attacker can construct and place their own transactions after seeing it. Every defence that hides the order attacks the wrong half of that: encrypted mempools, private relays and threshold schemes all try to stop the attacker seeing, which is hard, needs new infrastructure, and fails the moment one relay defects.

This attacks the other half. If the pool accepts only swaps that were committed to at least one block earlier, then an attacker who sees your order cannot act on it in the pool, because their own reaction would need a commitment made before they knew there was anything to react to. Seeing the order stops being useful.

Nothing has to be hidden, and nothing has to be trusted. A commitment is a hash of the swap's parameters and a salt, so it reveals nothing until it is used. Revealing means simply making the swap: the hook recomputes the hash from what actually arrived and requires it to match a commitment that is old enough, unused and unexpired.

A commitment that does not match is not a swap this pool will accept. Commitments carry a bond, and this is the part that makes the scheme cost something to abuse. Without it, a trader would commit to every order they might want at every size, then reveal the one they liked, which is a free option on the pool paid for by nobody.

The bond is returned in full when the commitment is used and forfeited to the pool when it expires unused, so keeping options open is exactly as expensive as the options are worth to you. The cost of all this is two transactions and a delay of at least one block, and that is the honest trade. A pool wearing this hook is not for the impatient; it is for flow large enough that being sandwiched costs more than waiting a block.

## Prior art

Commit-reveal is old, and appears on-chain in ENS registration, in sealed-bid auctions and in the sandwich-resistant AMM designs that reorder within a block. Encrypted mempools (Shutter, SUAVE, threshold schemes) attack the same problem by hiding the order. Requiring every swap in a pool to have been committed in an earlier block, so that seeing an order is useless rather than impossible, is the contribution here.

## Where it does not help

Two transactions and a block of delay on every trade, which rules the pool out for anything latency-sensitive and makes it unusable through routers that will not forward the salt. It does not stop an attacker who commits speculatively every block and reveals when they see something worth reacting to: the bond makes that expensive rather than impossible, and a pool that sets the bond too low is not protected, it is inconvenient. Finally, a reveal carries its salt in the clear, so an observer can copy it and execute your trade ahead of you. That is a denial rather than a theft, since the copier pays for the swap and the bond still returns to whoever committed, but a trader who is being griefed this way has no recourse beyond committing again.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    CommitRevealDarkHook.Config({
        delayBlocks: /* uint32 */ 0,
        ttlBlocks: /* uint32 */ 0,
        bond: /* uint128 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `delayBlocks` | `uint32` |  |
| `ttlBlocks` | `uint32` |  |
| `bond` | `uint128` |  |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AlreadyCommitted()` | This commitment already exists. Vary the salt. |
| `Expired(uint256)` | The commitment has expired and its bond is forfeited. |
| `InvalidDelay()` | A delay of zero would let somebody commit and reveal in the same block, which is no protection at all. |
| `InvalidTtl()` | The time to live must exceed the delay, or a commitment expires before it may be used. |
| `NotCommitted(bytes32)` | The swap does not match any standing commitment. |
| `NotExpiredYet(uint256)` | The commitment has not expired, so its bond cannot be swept. |
| `NothingToWithdraw()` | There is nothing to withdraw. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `TooSoon(uint256)` | The commitment is not old enough to be revealed yet. |
| `WrongBond(uint256)` | The bond sent does not match what the pool requires. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # CommitRevealDark
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # mev, anti-sandwich, commit-reveal, order-flow, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/commit-reveal-dark
cd commit-reveal-dark
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

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/commit-reveal-dark&type=Date)](https://www.star-history.com/#nirholas/commit-reveal-dark&Date)

<!-- three.ws:growth -->
## Support the project

If commit-reveal-dark saves you time, **[star it on GitHub](https://github.com/nirholas/commit-reveal-dark)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=commit-reveal-dark%3A%20Every%20swap%20must%20have%20been%20committed%20to%20in%20an%20earlier%20block%2C%20so%20nobody%20can%20react%20to%20an%20order%20they&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcommit-reveal-dark) · [Share on Bluesky](https://bsky.app/intent/compose?text=commit-reveal-dark%3A%20Every%20swap%20must%20have%20been%20committed%20to%20in%20an%20earlier%20block%2C%20so%20nobody%20can%20react%20to%20an%20order%20they%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fcommit-reveal-dark) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcommit-reveal-dark) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcommit-reveal-dark&t=commit-reveal-dark%3A%20Every%20swap%20must%20have%20been%20committed%20to%20in%20an%20earlier%20block%2C%20so%20nobody%20can%20react%20to%20an%20order%20they) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcommit-reveal-dark&title=commit-reveal-dark%3A%20Every%20swap%20must%20have%20been%20committed%20to%20in%20an%20earlier%20block%2C%20so%20nobody%20can%20react%20to%20an%20order%20they)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/commit-reveal-dark` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/commit-reveal-dark/issues) or [start a discussion](https://github.com/nirholas/commit-reveal-dark/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/commit-reveal-dark)](https://github.com/nirholas/commit-reveal-dark/graphs/contributors)

<!-- /three.ws:growth -->
