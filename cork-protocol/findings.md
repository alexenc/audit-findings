### M-1 Underflow risk in swapDsforRa and swapRaforDsin DsFlashSwap.sol :getAmountOutSellDs causing DOS attack

## Summary

The getAmountOutSellDS function in the DsFlashSwap

```solidity
 amountOut = amount - repaymentAmount;
```

library has a potential underflow vulnerability. This occurs when the repaymentAmount exceeds the amount parameter, leading to a panic error when attempting to compute amountOut. This situation can arise during the calculation of repayment amounts based on the reserves, particularly when the reserves are imbalanced.

That can lead to a Dos attack

## Root cause

Conditions for repaymentAmount > amount

1. Insufficient Reserves: If the reserves of raReserve and ctReserve are imbalanced, particularly if ctReserve is significantly lower than amount, the calculation may yield a high repaymentAmount.
2. High Amount Input: If the amount (the CT amount being sold) is large relative to the available reserves, it can lead to a situation where the required repayment amount exceeds the input amount.
3. Market Conditions: If the market conditions (e.g., price slippage, liquidity) are such that the amount of RA that needs to be borrowed to cover the CT being sold is disproportionately high, this can also lead to a higher repaymentAmount.

### H-1 Lack of slippage protection can lead to loss of funds

## Summary
There is no slippage protection while removing liquidity and swap tokens from AMM.

## Vuln detail

Lv token holder redeem before expiry `vaultLib::redeemEarly` is called it performs a swap of ct tokens for ra tokens in the amm, that function has no slippage protection as we can see the function's second parameter being hardcoded to 0.
That can lead to MEV problems making the user transacion to suffer from heavy slippage leading to potential loss of funds.

```solidity
ra = ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
```

## Recommendation

Protocol should implement slipagge protection when interacting with the amm.
