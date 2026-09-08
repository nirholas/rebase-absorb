# RebaseAbsorb

**A pool whose reserves are real tokens it holds itself, so anything that happens to those tokens happens to the people who supplied them.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://rebase-absorb.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/RebaseAbsorbHook.sol`](src/hooks/RebaseAbsorbHook.sol)
- **Licence:** Apache-2.0

## How it works

Uniswap v4 keeps every pool's tokens in one singleton and tracks ownership as ERC-6909 claims. That is what makes it cheap, and it has a consequence nobody has been able to work around: a claim does not rebase. If a rebasing token gains supply, the gain lands on the singleton's balance, where it belongs to no pool in particular and cannot be attributed to one, because the singleton holds the same token for every pool that trades it.

The same is true of a dividend paid to holders, and of an airdrop. The value arrives and strands. That is a property of where the tokens sit, not of v4.

A hook serving one pool is not a singleton. If the reserves live at the hook's own address, every one of those events is unambiguous: there is exactly one pool it could belong to, and its providers own it in proportion to their shares, with no accounting to do at all. The reserve is simply read as the balance, so a rebase up is already in the next quote and a rebase down is already priced in.

Nothing has to notice. Holding real tokens costs more gas per swap than holding claims, and that is the whole trade. It is the right trade for exactly the tokens the claims model cannot serve.

The same measurement makes fee-on-transfer tokens work, which is a second thing v4 cannot do. Because the hook quotes from what it actually received rather than from what the trader said they sent, a token that taxes its own transfers is priced correctly with no configuration, no oracle, and nothing to keep up to date when the token changes its own fee.

## Prior art

Wrappers are the standard answer: wstETH wraps a rebasing token into a fixed-balance one, and every AMM that supports fee-on-transfer tokens (Uniswap v2's supporting-fee router onwards) does so by measuring balances in the router. On v4 the received wisdom is that rebasing tokens need a wrapper, because the singleton cannot attribute a supply change to a pool. Moving the reserves to a per-pool hook address, so that the attribution question does not arise and the balance itself is the reserve, is the contribution here. It also covers what this catalogue listed separately as DividendPassthrough, since a dividend paid to holders is the same event as a rebase from the pool's point of view.

## Where it does not help

Real transfers on every swap, which is more gas than the claims model and the reason v4 does not do this by default. Exact-output swaps are refused outright rather than approximated: a taxed transfer cannot deliver an exact amount to the recipient, so promising one would be a lie the contract could not keep. A taxed input needs a router that over-sends, as explained above, and is outside what a hook can fix. One swap's input is also held as a claim until the next swap or a `sweep` converts it, and a rebase that lands in that window applies to everything except that amount. Reading the balance as the reserve also means a direct transfer to the hook is a donation to every share, which is harmless but is a way to move share value without a deposit. A token that can seize or freeze balances can do so here, exactly as it can anywhere else it is held. And only the pool's own two tokens are absorbed; an airdrop of a third token sits at the hook address with no way to reach anybody.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AddLiquidityThroughHook()` | Liquidity belongs to the hook, not to the pool, so the pool's own path is closed. |
| `AlreadyBound()` | This hook serves one pool, bound the first time one initializes with it. |
| `AmountTooSmall()` | The deposit was too small to mint a share, or the withdrawal too small to return anything. |
| `CallbackNotPoolManager()` | Only the `PoolManager` may drive the unlock callback. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExactOutputUnsupported()` | An exact-output swap cannot promise an exact receipt when the token taxes its own transfers. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InvalidFee()` | A fee at or above the whole trade is not a fee. |
| `NoReserves()` | The pool has no reserves to quote against yet. |
| `SafeCastOverflowedIntDowncast(uint8,int256)` | Value doesn't fit in an int of `bits` size. |
| `SafeCastOverflowedUintToInt(uint256)` | A uint value doesn't fit in an int of `bits` size. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `beforeInitialize`
- `beforeAddLiquidity`
- `beforeRemoveLiquidity`
- `beforeSwap`
- `beforeSwapReturnsDelta`

Mask: `0x2a88`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # RebaseAbsorb
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # curve, rebasing, fee-on-transfer, custom-curve, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/rebase-absorb
cd rebase-absorb
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
