# Goat Trading Security Review - 2024/08/09
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
The [goat-trading-dojo](https://github.com/inedibleX/goat-trading-dojo) repository was audited at commit [a73f8a245c5dd534db0d50de63921ac9fc962069](https://github.com/inedibleX/goat-trading-dojo/commit/a73f8a245c5dd534db0d50de63921ac9fc962069) 

The following contracts were in scope:
* src/FeeManager.sol 
* src/BaseGoatFactory.sol
* src/GoatTokenFactory.sol
* src/GoatTokenFactory1.sol
* src/LimitOrder.sol
* src/TaxToken.sol


# Findings Summary 
| ID           | Title                                                                                                                                                        | Severity | Status  |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |  -------- | ------- |
| [H-01](#H01) | Exploit in `LimitOrder.createOrder` Function Allows Fee Theft | HIGH | Fixed |
| [M-01](#M01) | Slippage applied in the `_removeLiquidity` function is useless | MEDIUM | Fixed |
| [M-02](#M02) | Risk of using `transfer` to send Eth | MEDIUM | Acknowledged |
| [L-01](#L01) | Redundant approval of the initialized position for feeManager from the factory contract | LOW | Fixed |
| [L-02](#L02) | Function `BaseGoatTokenFactory.generateSalt()` Should Implement Boundaries for Salt Generation | LOW | Acknowledged |
| [L-03](#L03) | Function `LimitOrder.fulfillOrders()` Should Validate Equal Lengths of `_users` and `_nftIds` | LOW | Acknowledged |
| [L-04](#L04) | `_removeLiquidity` Function Should Adhere to CEI Pattern | LOW | Fixed |

# Detailed Findings

## <a id="High"></a>High

### <a id="H01"></a> [H-01] Exploit in `LimitOrder.createOrder` Function Allows Fee Theft

#### Description
The `LimitOrder.createOrder()` function is intended to allow users to create new limit orders. In this function, there is a conditional check on line 112 meant to ensure that users create a limit order with `token1 == WETH`.

```solidity
function createOrder(INonfungiblePositionManager.MintParams memory params) public returns (uint256 nftId) {
    // Token 1 should always be WETH
    if (params.token0 == WETH && params.token1 != WETH) revert InvalidPair();
    
    ... 
}
```

However, this check is flawed due to the use of the `&&` operator instead of `||`.

As a result of this incorrect logic, an attacker can create an order with both `token0` and `token1` being different from `WETH` (e.g., `USDC` and `BTC`). This leads to the `LimitOrder` contract creating a position in the `USDC-BTC UniswapV3` pool, transferring the `USDC` and `BTC` tokens from the `LimitOrder` contract to the pool. Since the `createOrder()` function only pulls `token0 = USDC` from the user, the `token1 = BTC` is taken from the `LimitOrder` contract itself. This `BTC` may come from fees collected in positions in other pools (e.g., `BTC-WETH`), resulting in a loss of fees for the contract.
#### Impact
The contract may lose collected fees.
#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/LimitOrder.sol#L110-L112
#### Recommendation
Update the check on line 112 as follows:

```diff
function createOrder(INonfungiblePositionManager.MintParams memory params) public returns (uint256 nftId) {
    // Token 1 should always be WETH
-    if (params.token0 == WETH && params.token1 != WETH) revert InvalidPair();
+    if (params.token0 == WETH || params.token1 != WETH) revert InvalidPair();
```

#### Discussion
Protocol team: Fixed 

---

## <a id="Medium"></a>Medium

### <a id="M01"></a> [M-01] Slippage applied in the `_removeLiquidity` function is useless
#### Description
In the `_removeLiquidity()` function, `params.amount0Min` and `params.amount1Min` are calculated right before removing the liquidity of the position.
```solidity=
(params.amount0Min, params.amount1Min) = LiquidityAmounts.getAmountsForLiquidity(
    sqrtPriceX96, sqrtPriceAX96, sqrtPriceBX96, params.liquidity - params.liquidity / SLIPPAGE
);
params.deadline = block.timestamp;
positionManager.decreaseLiquidity(params);
```
The issue is that during this transaction, the tick of the Uniswap v3 pool cannot change. This means that the token amounts received from removing liquidity will exactly match the results of the `getAmountsForLiquidity()` function, given the specific `params.liquidity` amount. Consequently, `amount0Min` and `amount1Min` are always less than the actual token amounts received
#### Impact
There will be no slippage protection when removing liquidity from Uniswap v3 pool.
#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/LimitOrder.sol#L215-L220
#### Recommendation
For the `fulfillOrders` function, there is no need to add a slippage condition because it returns WETH, and pool fluctuations do not affect the received amount. However, for the `cancelOrder` function, slippage should be accounted for in the function's parameters.
#### Discussion
Protocol team: Fixed

### <a id="M02"></a> [M-02] Risk of using `transfer` to send Eth
#### Description
There are multiple instances in the code that send Eth using the transfer() function.
Sending Eth with the transfer function forwards a fixed amount of gas: 2300.
If the receiver is a smart contract and it utilizes more than 2300 gas in the fallback, it will revert. Therefore, it should use call instead of transfer to support smart contract receivers.
[Consensys Blog: Stop Using Solidity's transfer Now!](https://consensys.io/diligence/blog/2019/09/stop-using-soliditys-transfer-now/ )
#### Impact
Sending Eth may revert in cases where the receiver is a smart contract.
#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/LimitOrder.sol#L161
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/LimitOrder.sol#L235
#### Recommendation
Should use `call` instead of `transfer` to send Eth
#### Discussion
Protocol team: Acknowledged

## <a id="Low"></a>Low 

### <a id="L01"></a> [L-01] Redundant approval of the initialized position for feeManager from the factory contract
#### Description
In the `BaseGoatTokenFactory::_createPoolAndAddLiquidity()` function, after initializing and minting the first position, it also approves that position's NFT to the feeManager contract to collect fees. However, there is no direct fee collection call to the PositionManager in the FeeManager contract, meaning it never collects fees on behalf of the factory or uses that approval. The `collect()` function of the BaseGoatTokenFactory contract already calls `collect()` on the PositionManager as the owner of the positions, making that approval unnecessary.
#### Code snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/BaseGoatTokenFactory.sol#L160-L162
#### Recommendation
That approval should be removed
#### Discussion
Protocol team: Fixed

### <a id="L02"></a> [L-02] Function `BaseGoatTokenFactory.generateSalt()` Should Implement Boundaries for Salt Generation
#### Description
The `BaseGoatTokenFactory.generateSalt()` function generates a unique salt for token deployment by iterating `i` from `0` until the `salt` produces a valid `token` address.

This approach can lead to issues if the first valid salt is a very large number, potentially causing the transaction to revert due to running out of gas.

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/BaseGoatTokenFactory.sol#L229-L241

#### Recommendation
Consider adding lower and upper bounds for the salt to prevent excessive iteration:

```diff
- function generateSalt(address deployer, TokenType tokenType, bytes calldata constructorArgs)
+ function generateSalt(address deployer, TokenType tokenType, bytes calldata constructorArgs, uint lowerSalt, uint upperSalt)
    external
    view
    returns (bytes32 salt, address token)
{
-    for (uint256 i;; i++) {
+    for (uint256 i = lowerSalt; i <= upperSalt; i++) {
        salt = bytes32(i);
        token = predictTokenAddress(deployer, tokenType, salt, constructorArgs);
        if (token < WETH && token.code.length == 0) {
            break;
        }
    }
}
```
#### Discussion
Protocol team: Acknowledged

### <a id="L03"></a> [L-03] Function `LimitOrder.fulfillOrders()` Should Validate Equal Lengths of `_users` and `_nftIds`
#### Description
The `LimitOrder.fulfillOrders()` function is designed to fulfill multiple orders at once. However, it lacks a check to ensure that the lengths of `_users` and `_nftIds` arrays are equal, which could lead to a reverted transaction.

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/LimitOrder.sol#L144-L149

#### Recommendation
Add a requirement to verify that the `_users` and `_nftIds` arrays have the same length:

```diff
+ error LengthMismatch();

function fulfillOrders(address payable[] memory _users, uint256[] memory _nftIds) public {
+        if (_users.length != _nftIds.length) revert LengthMismatch();
        
        uint256 length = _users.length;
        for (uint256 i = 0; i < length; i++) {
            _removeLiquidity(_nftIds[i], _users[i], true);
        }
    }
```
#### Discussion
Protocol team: Acknowledged

### <a id="L04"></a> [L-04] `_removeLiquidity` Function Should Adhere to CEI Pattern
#### Description
The `_removeLiquidity()` function is tasked with removing liquidity from a position. Currently, the function transfers tokens to users (at lines 235 and 240) before deleting the order's information (at line 243).

This sequence introduces a potential vulnerability to reentrancy attacks, where a user could exploit the callback of an ETH transfer (at line 235) to manipulate the still-existing information of an deleted nft. 

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_08_09/blob/a73f8a245c5dd534db0d50de63921ac9fc962069/src/LimitOrder.sol#L231-L243

#### Recommendation
Consider deleting the `_orders[user][nftId]` entry before transferring tokens to the user:

```diff
function _removeLiquidity(uint256 nftId, address payable user, bool fulfill) internal {
    ... 
    
+    delete _orders[user][nftId];

    // convert to ETH
    if (amount1 > 0) {
        IWETH(WETH).withdraw(amount1);
        amount1 -= amount1Fees;
        user.transfer(amount1);
    }

    if (amount0 > 0) {
        amount0 -= amount0Fees;
        IERC20(order.token).safeTransfer(user, amount0);
    }
    
    emit OrderCompleted(user, nftId, order.pool);
-    delete _orders[user][nftId];
}
```
#### Discussion
Protocol team: Fixed