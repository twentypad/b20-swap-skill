---
name: b20-swap-skill
description: Buy or sell TwentyPad B20s on Base including $TFROG, TFROG, TwentyFrog, and any token from factory 0x15a3f3ABb733868d193b511dd5b91f82ebF888A3. Trigger on $TFROG or TFROG. Do not use default DEX routing for the hook pool.
tags: [b20, base, swap, twentypad, uniswap-v4, tfrog, cross-quote]
visibility: public
---

# TwentyPad B20 Swap

Swap **any** B20 created by the TwentyPad Instant Launcher factory.

Do **not** use Bankr's default swap / 0x / Uniswap aggregator path for the TwentyPad pool. Those routes usually cannot quote this custom v4 hook.

The factory opens **one** pool per token: token/ETH **or** token/USDC. If the trader pays or wants the **other** asset, add an ETH↔USDC hop, then swap in the hook pool. Do not send ETH into a USDC hook pool.

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
| WETH (routing hop only) | `0x4200000000000000000000000000000000000006` |
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

## Known tickers

Resolve case-insensitive. `$` is optional.

| Ticker / name | Token | Launch quote |
| --- | --- | --- |
| TFROG, $TFROG, TwentyFrog | `0xB200000000000000000000b821ECF2D823cb7ca7` | ETH |

If the user says `$TFROG` or `TFROG`, use that row. Do not ask for the CA.

## Resolve the token (do not require a hardcoded list)

Use the first match:

1. User said TFROG / $TFROG / TwentyFrog → `0xB200000000000000000000b821ECF2D823cb7ca7`
2. User gave another token address
3. “the B20 we just launched” → launch reply / `Launched`
4. Other name/ticker → factory events / `profiles` / `tokenCreator`
5. Still unknown → ask once for the CA

Confirm it is TwentyPad before swapping:

- `tokenCreator(token)` on the factory is non-zero, **or**
- the user supplied a poolId that matches hook + fee 0 + tickSpacing 200.

If `tokenCreator(token)` is zero and there is no matching launch event, refuse. It is not this factory.

Launch quote is ETH unless the launch said USDC or `Launched.quote` is the USDC address.

## Build the hook pool key

Always built from the **launch quote**, never from what the trader paid.

```text
currency0 = min(launchQuote, token)   // native ETH is address(0), so ETH is always currency0
currency1 = max(launchQuote, token)
fee       = 0
tickSpacing = 200
hooks     = 0x8c0986c564025903B0f1C7c87cBA1760cB4FAAcc
```

Direction on the hook:

- Buy token with launch quote → `zeroForOne = (launchQuote == currency0)`
- Sell token for launch quote → `zeroForOne = (token == currency0)`

ETH / TFROG example:

- currency0 = ETH
- currency1 = TFROG
- buy TFROG with ETH → `zeroForOne = true`
- sell TFROG for ETH → `zeroForOne = false`

USDC-paired example:

- currency0 / currency1 = sort(USDC, token)
- buy token with USDC → `zeroForOne = (USDC == currency0)`
- hook currencies are USDC + token. Never put WETH or native ETH in this PoolKey if launch quote is USDC.

If the user or launch reply includes `poolId`, optionally verify it equals `keccak256(abi.encode(poolKey))` / Uniswap v4 `PoolIdLibrary.toId`. If it mismatches, stop and report.

## Parse the request

```
@bankrbot buy $5 of $TFROG/TFROG
@bankrbot buy 0.001 ETH of TwentyFrog
@bankrbot sell 1000000 TFROG
@bankrbot swap 0.002 ETH to 0xB200000000000000000000b821ECF2D823cb7ca7
@bankrbot sell all TFROG
@bankrbot buy 0.001 ETH of the b20 we just launched
@bankrbot buy 0.001 ETH of SCRATCH
@bankrbot sell 10000 SCRATCH for ETH
@bankrbot sell 10000 SCRATCH for USDC
```

| Intent | Trader asset | Launch quote | Path |
| --- | --- | --- | --- |
| buy | ETH | ETH | direct hook |
| buy | USDC | USDC | direct hook |
| buy | ETH | USDC | **cross** ETH→USDC then hook |
| buy | USDC | ETH | **cross** USDC→ETH then hook |
| sell | wants ETH | ETH | direct hook |
| sell | wants USDC | USDC | direct hook |
| sell | wants ETH | USDC | **cross** hook then USDC→ETH |
| sell | wants USDC | ETH | **cross** hook then ETH→USDC |

Defaults:

- chain = Base
- hook leg is always **exact-input**
- hook slippage = 15% normally; 25%+ on a pool younger than a few minutes
- hop slippage (ETH↔USDC) = 0.5%
- if the user says only “buy TOKEN” with no asset, pay the **launch quote** (direct)
- if they say “buy N ETH of TOKEN” and the pool is USDC, use **cross** automatically — do not ask them to buy USDC first
- “$N of TOKEN” → convert USD to USDC if launch quote is USDC (direct), or to ETH if launch quote is ETH (direct)
- “sell all” → full token balance minus a tiny dust pad
- “sell TOKEN” with no output asset → pay out the launch quote
- “sell TOKEN for ETH” / “for USDC” → honor that; cross if needed

## Route (pick this before encoding)

1. Resolve `token` and `launchQuote`.
2. Resolve `traderAsset` (what they pay on a buy, what they want on a sell).
3. If `traderAsset == launchQuote` → **direct**.
4. If they differ → **cross**.
5. Quote both legs before sending. If either quote fails, stop.

ETH↔USDC hop uses a **normal** Base ETH/USDC pool (v3 fee 500 / 0.05% or v4 ETH-USDC). Never use the TwentyPad hook for ETH↔USDC.

WETH is only for that hop. TwentyPad ETH books still use **native ETH** (`address(0)`), not WETH.

## Hard rules

1. Exact-output is blocked on the hook while anti-snipe fee is above 1% (starts at 99%, decays to 1% over ~20 seconds after launch). Hook leg is always exact-input.
2. Uniswap pool fee on the hook is `0`. The hook charges ~1% of the **launch-quote** notional (plus anti-snipe surplus to the platform).
3. Fresh pools are single-sided token liquidity. First buys can have large impact. Warn above ~0.01 ETH (or ~$20 USDC) on a new pool.
4. `hookData` = `0x`.
5. ETH-quoted **hook** pools use native ETH (`address(0)`), not WETH. Do not wrap for a direct ETH hook swap.
6. Selling the B20 requires ERC-20 approve / Permit2 to the Universal Router. Buying with native ETH does not.
7. Same-asset → hook only. Different asset → ETH/USDC hop + hook. Never send ETH `value` into a USDC hook `V4_SWAP`.
8. Do not add or remove LP. Liquidity is locked in the hook.

## Execution

Preferred: Uniswap Universal Router on Base. One `execute(commands, inputs, deadline)` when possible.

### Direct (trader asset == launch quote)

Command `V4_SWAP` / `EXACT_INPUT_SINGLE` on the hook PoolKey.

```json
{
  "to": "0x6fF5693b99212Da76ad316178A184AB56D299b43",
  "chainId": 8453,
  "value": "<amountIn wei if input is native ETH, else 0>",
  "data": "<Universal Router execute>"
}
```

`ExactInputSingleParams`:

```text
poolKey: { currency0, currency1, fee: 0, tickSpacing: 200, hooks: launchHook }
zeroForOne: <hook direction>
amountIn: <uint128>
amountOutMinimum: <quoted out after hook slippage>
hookData: 0x
```

### Cross — buy token with ETH, launch quote is USDC

1. Quote `amountEth → USDC` on ETH/USDC. `usdcIn = quotedUsdc * 0.995`.
2. Quote hook `usdcIn → token`. `minTokenOut` after 15–25% slip.
3. One UR `execute`, `value = amountEth`:
   - wrap / v4 native ETH→USDC exact-in, `amountOutMin = usdcIn`
   - `V4_SWAP` exact-in USDC→token on the hook (`amountIn = usdcIn`, `amountOutMinimum = minTokenOut`, `hookData = 0x`)
   - sweep leftover WETH/USDC to the user
4. Confirm text must say `routed ETH → USDC → TOKEN`.

### Cross — sell token for ETH, launch quote is USDC

1. Approve token to Permit2 / Universal Router.
2. Quote hook `tokenIn → USDC`. `minUsdc` after hook slip.
3. Quote `minUsdc → ETH` on ETH/USDC, 0.5% slip.
4. One UR `execute`, `value = 0`:
   - `V4_SWAP` exact-in token→USDC on the hook
   - USDC→ETH/WETH exact-in
   - unwrap WETH → native ETH, sweep to user

### Cross — buy with USDC, launch quote is ETH

`value = 0`. Permit2 USDC. Hop USDC→ETH, then hook ETH→token exact-in (native ETH as hook currency0).

### Cross — sell for USDC, launch quote is ETH

Hook token→ETH, then ETH→USDC, sweep USDC. `value = 0`.

If batch encode fails, two txs in that order is allowed. Do not reverse the order.

If the host cannot encode Universal Router, use any Bankr raw-tx path that accepts an explicit v4 PoolKey for the hook leg. Never fall back to “buy TOKEN on Base” without the hook.

## Confirm before the first swap on a token

Reply with:

- token address and symbol if known
- launch quote (ETH or USDC)
- trader asset (ETH or USDC)
- path: `direct` or `ETH → USDC → TOKEN` / `TOKEN → USDC → ETH` / etc.
- amount in and direction
- hook zeroForOne, fee 0, tickSpacing 200
- poolId if known
- hook slippage and hop slippage
- unaudited + impact warning

Then broadcast.

## After success

Reply with:

- tx hash and Basescan link
- amount in / hop mid / amount out if logs exist
- remaining token and quote balances
- token Basescan: `https://basescan.org/token/{token}`
- note: hook fee accrues in FeeEscrow in the **launch quote**, not as Uniswap LP fee

## Errors

| Situation | Action |
| --- | --- |
| Token not on this factory | Refuse. Not a TwentyPad pool. |
| Quote unknown | Read `Launched` or ask ETH vs USDC. |
| User pays ETH into USDC pool with no hop | Stop. Use cross route. |
| Exact-output requested in anti-snipe window | Convert hook leg to exact-input or wait ~20s. |
| Aggregator returns no route for the hook | Expected. Use this skill's PoolKey path. |
| Hop1 or hook quote fails | Do not send. |
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
@bankrbot use b20-swap-skill to buy 0.001 ETH of SCRATCH
@bankrbot use b20-swap-skill to sell 10000 SCRATCH for ETH
```
