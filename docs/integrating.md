# Integrating CommitRevealDark

## From Solidity

```solidity
import {CommitRevealDarkHook} from "commit-reveal-dark/src/hooks/CommitRevealDarkHook.sol";
import {PoolKey} from "@uniswap/v4-core/src/types/PoolKey.sol";

PoolKey memory key = PoolKey({
    currency0: currencyA,          // must sort below currency1
    currency1: currencyB,
    fee: 3000,
    tickSpacing: 60,
    hooks: IHooks(address(hook))
});

hook.configure(key, CommitRevealDarkHook.Config({ /* see the README */ }));
poolManager.initialize(key, startingSqrtPriceX96);
```

## From TypeScript

```ts
import {getHook, hookAddress, poolKeyFor, poolId} from "@hookforge/sdk";

const hook = getHook("commit-reveal-dark");
const address = hookAddress("commit-reveal-dark", 8453);              // Base
const key = poolKeyFor({
  hook: address,
  currencyA: USDC,
  currencyB: WETH,
  tickSpacing: 60,
});
console.log(poolId(key));
```

## Identifying the hook from an address

Any caller holding only a hook address can find out what it is, without a registry:

```solidity
IHookMetadata(hookAddress).hookName();   // "CommitRevealDark"
IHookMetadata(hookAddress).specURI();    // points at hook.json
```

The permission bits are in the address itself, so a caller can also decode what the hook participates in with no call at all:

```ts
import {permissionsOf} from "@hookforge/sdk";
permissionsOf(address);
```

## Things that will bite you

- **Configure before you initialize.** The parameters are fixed for the pool's lifetime, and a pool the hook was never configured for cannot be created at all.
- **Anyone can configure a key that has no pool yet.** If someone front-runs your configuration with terms you did not want, pick a different `tickSpacing` and configure that instead. It is a different pool id and it costs them their gas.
- **This hook reads `hookData`.** A router or aggregator that strips it cannot trade the pool.
- **Deterministic addresses are not deployments.** An address published before a deploy is where the hook *will* be. There is no code at it until the deploy runs.

More at https://commit-reveal-dark.pages.dev.
