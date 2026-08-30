---
name: b20-swap-skill
description: Buy or sell any TwentyPad Instant Launcher B20 on Base Uniswap v4. Use when the user wants to swap TFROG, TwentyFrog, a TwentyPad B20, a token from factory 0x15a3f3ABb733868d193b511dd5b91f82ebF888A3, or any B20 whose pool uses hook 0x8c0986c564025903B0f1C7c87cBA1760cB4FAAcc. Do not use Bankr default DEX routing for these tokens.
tags: [b20, base, swap, twentypad, uniswap-v4, tfrog]
version: 2
visibility: public
metadata:
  clawdbot:
    emoji: "🔄"
    homepage: "https://github.com/twentypad/b20-instant-launcher"
---

# TwentyPad B20 Swap

Swap **any** B20 created by the TwentyPad Instant Launcher factory.

Do **not** use Bankr's default swap / 0x / Uniswap aggregator path. Those routes usually cannot quote this custom v4 hook.

TFROG is an example, not a whitelist. If the token is missing from this file, resolve it on-chain or from the user's launch reply, then swap with the same pool key recipe.

## Fixed TwentyPad parameters (every launch)

| Piece | Value |
| --- | --- |
| Chain | Base (`8453`) |
| Factory | `0x15a3f3ABb733868d193b511dd5b91f82ebF888A3` |
| Launch hook | `0x8c0986c564025903B0f1C7c87cBA1760cB4FAAcc` |
| Fee escrow | `0xD43586103c760Bd5e139a2De2655413dE441B150` |
| PoolManager | `0x498581fF718922c3f8e6A244956aF099B2652b2b` |
| Universal Router | `0x6fF5693b99212Da76ad316178A184AB56D299b43` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| Pool fee | `0` |
| tickSpacing | `200` |
| ETH quote | `0x0000000000000000000000000000000000000000` |
| USDC quote | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |

Site: https://twentypad.com  
Repo: https://github.com/twentypad/b20-instant-launcher  
Contracts are unaudited.

## Example launch (not required)

From https://x.com/bankrbot/status/2094147138016186772

| Field | Value |
| --- | --- |
| Name / symbol | TwentyFrog / TFROG |
| Token | `0xB200000000000000000000b821ECF2D823cb7ca7` |
| Quote | ETH |
| PoolId | `0x1d68d144cdf428dfb24c60e0136637ab451344ec1bc18d8f8b0a08294d500f2a` |
| Creator | `0xd4aedc4595196305a60d8ef2dd9b9ba27021b4cb` |

Use this row only when the user means TFROG / TwentyFrog. For every other TwentyPad token, resolve dynamically.

## Resolve the token (do not require a hardcoded list)

Use the first match:

1. User gave a token address.
2. User said TFROG / TwentyFrog → use the example row above.
3. User said “the B20 we just launched” / replied under a launch tweet → take `token`, `quote`, and `poolId` from that Bankr launch reply or from factory `Launched`.
4. User gave a name/ticker only → look up recent `Launched` events on the factory, or factory `profiles(token)` / `tokenCreator(token)`.
5. Still unknown → ask once for the token address or launch tx. Do not invent an address.

Confirm it is TwentyPad before swapping:

- `tokenCreator(token)` on the factory is non-zero, **or**
- the user supplied a poolId that matches hook + fee 0 + tickSpacing 200.

If `tokenCreator(token)` is zero and there is no matching launch event, refuse. It is not this factory.

Quote is ETH unless the launch said USDC or `Launched.quote` is the USDC address.

## Build the pool key

```text
currency0 = min(quote, token)   // native ETH is address(0), so ETH is always currency0
currency1 = max(quote, token)
fee       = 0
tickSpacing = 200
hooks     = 0x8c0986c564025903B0f1C7c87cBA1760cB4FAAcc
```

Direction:

- Buy token with quote → `zeroForOne = (quote == currency0)`
- Sell token for quote → `zeroForOne = (token == currency0)`

ETH / TFROG example:

- currency0 = ETH
- currency1 = TFROG
- buy TFROG with ETH → `zeroForOne = true`
- sell TFROG for ETH → `zeroForOne = false`

If the user or launch reply includes `poolId`, optionally verify it equals `keccak256(abi.encode(poolKey))` / Uniswap v4 `PoolIdLibrary.toId`. If it mismatches, stop and report.

## Parse the request

```
@bankrbot buy $5 of TFROG
@bankrbot buy 0.001 ETH of TwentyFrog
@bankrbot sell 1000000 TFROG
@bankrbot swap 0.002 ETH to 0xB200000000000000000000b821ECF2D823cb7ca7
@bankrbot sell all TFROG
@bankrbot buy 0.001 ETH of the b20 we just launched
```

| Intent | Input | Output |
| --- | --- | --- |
| buy / long / ape | launch quote (ETH or USDC) | B20 token |
| sell | B20 token | launch quote |

Defaults:

- chain = Base
- only **exact-input**
- slippage = 15% normally; 25%+ on a pool younger than a few minutes
- “$N of TOKEN” → convert USD to quote using a spot price, then exact-in that quote amount
- “sell all” → full token balance minus a tiny dust pad if needed for gas on ETH

## Hard rules

1. Exact-output is blocked while anti-snipe fee is above 1% (starts at 99%, decays to 1% over ~20 seconds after launch). Always exact-input.
2. Uniswap pool fee is `0`. The hook charges ~1% of the **quote** notional (plus anti-snipe surplus to the platform).
3. Fresh pools are single-sided token liquidity. First buys can have large impact. Warn above ~0.01 ETH on a new pool.
4. `hookData` = `0x`.
5. ETH-quoted pools use **native ETH** (`address(0)`), not WETH. Do not wrap.
6. Selling the B20 requires ERC-20 approve / Permit2 to the Universal Router. Buying with native ETH does not.
7. Do not multi-hop through WETH or USDC. Swap only in the launch pool.
8. Do not add or remove LP. Liquidity is locked in the hook.

## Execution

Preferred: Uniswap Universal Router on Base, command `V4_SWAP`, `EXACT_INPUT_SINGLE`.

1. If selling the B20 and allowance is 0, approve the token to Permit2 and/or Universal Router.
2. Submit:

```json
{
  "to": "0x6fF5693b99212Da76ad316178A184AB56D299b43",
  "chainId": 8453,
  "value": "<amountIn wei if input is native ETH, else 0>",
  "data": "<Universal Router execute(commands, inputs, deadline)>"
}
```

`ExactInputSingleParams`:

```text
poolKey: { currency0, currency1, fee: 0, tickSpacing: 200, hooks: launchHook }
zeroForOne: <from direction above>
amountIn: <uint128>
amountOutMinimum: <amountIn-quoted out after slippage>
hookData: 0x
```

3. Wait for confirmation.

If the host cannot encode Universal Router, use any Bankr raw-tx path that accepts an explicit v4 PoolKey (hook + fee 0 + spacing 200). Never fall back to “buy TOKEN on Base” without the hook.

## Confirm before the first swap on a token

Reply with:

- token address and symbol if known
- quote (ETH or USDC)
- amount in and direction
- zeroForOne
- hook, fee 0, tickSpacing 200
- poolId if known
- slippage
- unaudited + impact warning

Then broadcast.

## After success

Reply with:

- tx hash and Basescan link
- amount in / amount out if logs exist
- remaining token and quote balances
- token Basescan: `https://basescan.org/token/{token}`
- note: hook fee accrues in FeeEscrow, not as Uniswap LP fee

## Errors

| Situation | Action |
| --- | --- |
| Token not on this factory | Refuse. Not a TwentyPad pool. |
| Quote unknown | Read `Launched` or ask ETH vs USDC. |
| Exact-output requested in anti-snipe window | Convert to exact-input or wait ~20s. |
| Aggregator returns no route | Expected. Use this skill's PoolKey path. |
| High impact / tiny out | Warn, reduce size, or raise slippage only if the user agrees. |

## What this skill does not do

- Launch tokens (use `b20-launcher-skill`)
- Claim creator/platform fees from FeeEscrow
- Add or remove liquidity
- Trade non-TwentyPad B20s or ERC-20s

## Example commands

```
@bankrbot use b20-swap-skill to buy 0.001 ETH of TFROG
@bankrbot use b20-swap-skill to sell all TFROG
@bankrbot use b20-swap-skill to buy 0.001 ETH of 0xB200000000000000000000b821ECF2D823cb7ca7
@bankrbot use b20-swap-skill to sell 500000 of the b20 we just launched
```
