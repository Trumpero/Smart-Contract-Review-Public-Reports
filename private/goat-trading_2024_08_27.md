# Goat Trading Security Review - 2024/08/27
###### tags: `private`, `goat-trading`

# Introduction 
A security review of the Goat Trading protocol was done by Trumpero team. 

This audit report includes all the vulnerabilities, issues and code improvements found during the security review.

# Disclaimer
A smart contract security review cannot assure the absolute absence of vulnerabilities. It involves a constrained allocation of time, resources, and expertise to identify as many vulnerabilities as possible. I cannot provide a guarantee of 100% security following the review, nor can I guarantee that any issues will be discovered during the review of your smart contracts.

# About Trumpero Team 
The Trumpero team was established by two independent smart contract researchers, Trungore and duc, who share a profound interest in Web3 security. Demonstrating their capabilities through numerous audits, contests, and bug bounties, the team is dedicated to contributing to the blockchain ecosystem and its protocols by investing significant time and effort into security research and reviews.

Twitter - [Trungore](https://twitter.com/Trungore), [duc](https://twitter.com/duc_hph) \
Sherlock - [Trumpero](https://audits.sherlock.xyz/watson/Trumpero) \
Code4rena - [KIntern_NA](https://code4rena.com/@KIntern_NA)

# Severity Classification
## Severity
| **Severity** | **Impact: High** | **Impact: Medium** | **Impact: Low** |
|---|---|---|---|
| **Likelihood: High** | Critical  | High | Medium |  
| **Likelihood: Medium** | High | Medium | Low |   
| **Likelihood: Low** | Medium | Low | Low |   

## Impact 
**High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.

**Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.

**Low** - can lead to any kind of unexpected behaviour with some of the protocol's functionalities that's not so critical.

## Likelihood

**High** - attack path is possible with reasonable assumptions that mimic on-chain conditions, and the cost of the attack is relatively low compared to the amount of funds that can be stolen or lost.

**Medium** - only a conditionally incentivized attack vector, but still relatively likely.

**Low** - has too many or too unlikely assumptions or requires a significant stake by the attacker with little or no incentive.

# Audit scope
The audit contains 2 audit scopes within 2 repositories 
* [v3-periphery-dojo](https://github.com/inedibleX/v3-periphery-dojo) at commit [f18910965b9f5c3ffc94e54a04b4ca18a12d1803](https://github.com/inedibleX/v3-periphery-dojo/tree/f18910965b9f5c3ffc94e54a04b4ca18a12d1803)
* [goat-trading-dojo](https://github.com/inedibleX/goat-trading-dojo) at commit [1c51f13ad6581e873e601e9a3f9abe390be78f40](https://github.com/inedibleX/goat-trading-dojo/tree/1c51f13ad6581e873e601e9a3f9abe390be78f40)

The following contracts were in scope:
* [v3-periphery-dojo/blob/main/contracts/SwapRouter.sol](https://github.com/inedibleX/v3-periphery-dojo/blob/f18910965b9f5c3ffc94e54a04b4ca18a12d1803/contracts/SwapRouter.sol)
* [goat-trading-dojo/blob/main/src/tokens/TaxToken.sol](https://github.com/inedibleX/goat-trading-dojo/blob/1c51f13ad6581e873e601e9a3f9abe390be78f40/src/tokens/TaxToken.sol)


# Findings Summary 
| ID           | Title                                                                                                                                                        | Severity | Status  |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |  -------- | ------- |
| [M-01](#M01) | `exactInputInternal()` may return the wrong amount of tokenOut, causing DoS for `exactInput()` | MEDIUM | Pending |
| [M-02](#M02) | Missing apply buy tax in `exactOutputInternal()` function may cause incorrect received amount or revert on `exactOutput()` | MEDIUM | Pending |

# Detailed Findings

## <a id="Medium"></a>Medium

### <a id="M01"></a> [M-01] `exactInputInternal()` may return the wrong amount of tokenOut, causing DoS for `exactInput()`
#### Description
In the SwapRouter contract, the `exactInputInternal()` function returns the result of `UniswapV3Pool.swap()` as the token amount out.
```solidity=
  (int256 amount0, int256 amount1) =
      getPool(tokenIn, tokenOut, fee).swap(
          recipient,
          zeroForOne,
          amountIn.toInt256(),
          sqrtPriceLimitX96 == 0
              ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
              : sqrtPriceLimitX96,
          abi.encode(data)
      );
  
  return uint256(-(zeroForOne ? amount1 : amount0));
```
However, this amountOut may not be equal to the amount of tokenOut received from the swap. This is because `UniswapV3Pool.swap()` only returns the amount of tokens before transferring, and tokenOut can be a tax token with a buy tax from the pair. In this case, because a buy tax is charged during transferring, the `exactInputInternal()` function will return an amount of tokens received that is higher than the actual amount.
#### Impact
During the `exactInput()` function, if `exactInputInternal()` returns an incorrect amount of tokens received (higher than the actual received amount), it can cause the next swap in the path to revert. This happens because the incorrect amount out is used for the next swap, and the actual received tokens are insufficient. Therefore, this issue may lead to a DoS for the `exactInput()` function if there is a tax token in the path.
#### Code Snippet
https://github.com/inedibleX/v3-periphery-dojo/blob/f18910965b9f5c3ffc94e54a04b4ca18a12d1803/contracts/SwapRouter.sol#L121-L132
#### Recommendation
The `exactInputInternal()` function should return the increased amount of tokenOut balance
#### Discussion
Protocol team: ???

### <a id="M02"></a> [M-02] Missing apply buy tax in `exactOutputInternal()` function may cause incorrect received amount or revert on `exactOutput()`
#### Description
In the SwapRouter contract, the `exactOutputInternal()` function returns `amountOutReceived`, which is the result of `UniswapV3Pool.swap()`, as the token amount out.
```solidity=
  (int256 amount0Delta, int256 amount1Delta) =
    getPool(tokenIn, tokenOut, fee).swap(
        recipient,
        zeroForOne,
        -amountOut.toInt256(),
        sqrtPriceLimitX96 == 0
            ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
            : sqrtPriceLimitX96,
        abi.encode(data)
    );
  
  uint256 amountOutReceived;
  (amountIn, amountOutReceived) = zeroForOne
    ? (uint256(amount0Delta), uint256(-amount1Delta))
    : (uint256(amount1Delta), uint256(-amount0Delta));
  // it's technically possible to not receive the full output amount,
  // so if no price limit has been specified, require this possibility away
  if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);
```
However, this amountOut may not be equal to the amount of tokenOut received from the swap. When tokenOut is a tax token with a buy tax from the pair, it will incur a buy tax during the swap, causing the received amount to be smaller than the expected amount. 
The root cause of this issue is that `exactOutputInternal()` uses the expected received amount as the amountOut for `UniswapV3Pool.swap()`, without considering the buy tax when swapping from WETH to a tax token (zeroForOne is false).
#### Impact
During multiple swaps with `exactOutput()`, the calculation of the input amount will be incorrect because it does not apply the buy tax of the tax token. This can cause the path to revert if there is a tax token that charges a buy tax along the path. Additionally, `exactOutput()` and `exactOutputSingle()` exhibit incorrect behavior since they may not provide users with the expected amount of tokenOut.
#### Code Snippet
https://github.com/inedibleX/v3-periphery-dojo/blob/f18910965b9f5c3ffc94e54a04b4ca18a12d1803/contracts/SwapRouter.sol#L222-L257
https://github.com/inedibleX/v3-periphery-dojo/blob/f18910965b9f5c3ffc94e54a04b4ca18a12d1803/contracts/SwapRouter.sol#L136-L165
#### Recommendation
Buy tax for a tax token should be applied in the case of swapping an exact output from WETH to a tax token. Here is an example of the fix:
```solidity=
function applyTax(
    ...
    if (zeroForOne) {
      ...
    } else {
          address token = tokenIn < tokenOut ? tokenIn : tokenOut;
          bytes4 selector = bytes4(keccak256('getTaxes(address)'));
          (bool success, bytes memory data) =
              token.staticcall(abi.encodeWithSelector(selector, address(getPool(tokenIn, tokenOut, fee))));

          if (success && data.length >= 64) {
              (uint256 buyTax, ) = abi.decode(data, (uint256, uint256));

              if (!isExactInput) {
                  uint256 remainder = (amount * DIVISOR) % (DIVISOR - buyTax);
                  amount = (amount * DIVISOR) / (DIVISOR - tax);

                  if (remainder != 0) {
                      amount += 1;
                  }
              }
          }
    }
}

function exactOutputInternal(
    ...
    amountOut = applyTax(tokenIn, tokenOut, fee, amountOut, false, zeroForOne);
    ...
}
```
#### Discussion
Protocol team: ???


