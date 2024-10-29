# Goat Trading Security Review - 2024/10/23
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

The [goat-trading-dojo](https://github.com/inedibleX/goat-trading-dojo/tree/token-batch-1) repository was audited at commit [53197145fdce52b2d8a5ff293458970d6499b735](https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/commit/53197145fdce52b2d8a5ff293458970d6499b735) 

The following contracts were in scope:
* contracts/tokens/FundMeToken.sol 
* contracts/tokens/VestingLaunchToken.sol 

# Findings Summary 
| ID           | Title                                                                                                                                                        | Severity | Status  |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |  -------- | ------- |
| [H-01](#H01) | Users May Lose `VestingLaunchToken` When Using the `LimitOrder` Contract During the Vesting Period | HIGH | Fixed |
| [L-01](#L01) | Function `balanceOf()` May Underflow | LOW | Acknowledged |
| [L-02](#L02) | Function `balanceOf()` May Underflow | LOW | Acknowledged |
| [L-03](#L03) | Missing `TaxesUpdated` Event Emission When `funding >= fundingCap` | LOW | Acknowledged |
| [L-04](#L04) | Redundant Storage Variable `dispatcher` in `FundMeToken` Contract | LOW | Acknowledged |

# Detailed Findings

### <a id="H01"></a> [H-01] Users May Lose `VestingLaunchToken` When Using the `LimitOrder` Contract During the Vesting Period
#### Description
Consider a scenario where users utilize the `LimitOrder` contract to sell their `VestingLaunchToken` within the vesting period. If an order is canceled before being fulfilled, the `LimitOrder._removeLiquidity()` function transfers the unfulfilled `VestingLaunchToken` back to the order's owner.

```solidity
function _removeLiquidity(uint256 nftId, address payable user, bool fulfill) internal {
    ... 
    
    uint256 tokenBalBefore = IERC20(order.token).balanceOf(address(this));
    (uint256 amount0, uint256 amount1) = positionManager.collect(collectParams);
    
    ...

    if (amount0 > 0) {
        uint256 tokenBalAfter = IERC20(order.token).balanceOf(address(this));
        amount0 = tokenBalAfter - tokenBalBefore;
        amount0 -= amount0Fees;
        IERC20(order.token).safeTransfer(user, amount0);
    }
    emit OrderCompleted(user, nftId, order.pool);
}
```

However, this action increases the `amountDisbursed` for the `LimitOrder` contract itself, resulting in subsequent calls to `balanceOf()` returning a smaller balance of `VestingLaunchToken` than the contract actually holds. This discrepancy directly impacts all subsequent `cancelOrder()` transactions executed within the vesting period.

The issue occurs because the `amount0` calculated from the difference between the `LimitOrder` contract's balance before and after the `positionManager.collect(collectParams)` call in [line 310](https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/LimitOrder.sol#L310) is less than the actual amount of tokens received, due to the `amountDisbursed[LimitOrder] > 0`.

As a result, `VestingLaunchToken` can become stuck in the contract, causing the order's creator to lose tokens when the order is canceled.

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/tokens/VestingLaunchToken.sol#L53-L58

https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/LimitOrder.sol#L308-L313

#### Recommendation
Consider preventing users from using the `LimitOrder` contract to sell their tokens during the `vestingPeriod`.

#### Discussion
Protocol team: fixed

### <a id="L01"></a> [L-01] Function `balanceOf()` May Underflow

#### Description
The `VestingLaunchToken.balanceOf()` function retrieves the balance of a user, considering any active vesting period. If the vesting period has passed, the function returns `_balances[user]`. Otherwise, the balance is calculated as follows:

- Let `B` be `balances[user]`
- Let `A` be `amountDisbursed[user]`
- Let `delta = block.timestamp - vestingStart`

The formula for balance during the vesting period is:

```
(B + A) * delta / vestingPeriod - A
= B * delta / vestingPeriod - A * (vestingPeriod - delta) / vestingPeriod
```

This calculation can result in an underflow if `B * delta / vestingPeriod` is less than `A * (vestingPeriod - delta) / vestingPeriod`, which leads to a negative balance, causing the function to revert due to an underflow error.

For example:

1. Given the state:
   - `vestingStart = 0`
   - `vestingEnd = 100`
2. Alice holds `100` tokens:
   - `balances[Alice] = 100`
3. Alice transfers `90` tokens to Bob:
   - `balances[Alice] = 10`
   - `amountDisbursed[Alice] = 90`
4. At `block.timestamp = 10`:
   - `balanceOf(Alice) = (10 + 90) * 10 / 100 - 90 = -80` (underflow error).

This issue prevents the `balanceOf()` function from being called successfully.

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/tokens/VestingLaunchToken.sol#L45-L59

#### Recommendation
To avoid underflow, modify the `balanceOf()` function as follows:

```diff
function balanceOf(address user) public view override returns (uint256 balance) {
    balance = _balances[user];
    /** @dev Vesting start exception is so pool can be setup by owner.
     *       Theoretically if multiple people snipe on this block one could
     *       sell everything to sandwich but meh.
     *
     *       Permanent pool exception so it can sell freely.
    */ 
    if (block.timestamp < vestingEnd && block.timestamp != vestingStart && user != pool) {
        // amountDisbursed here is needed so % calculations are consistent throughout period.
        balance += amountDisbursed[user];
        balance = balance * (block.timestamp - vestingStart) / vestingPeriod;
+        if (balance <= amountDisbursed[user]) {
+            balance = 0;  
+        }
+        else {
            balance -= amountDisbursed[user];
+        }
    }
}
```

#### Discussion
Protocol team: Acknowledged

### <a id="L02"></a> [L-02] Missing `TaxesUpdated` Event Emission When `funding >= fundingCap`
#### Description
The `TaxToken.TaxesUpdated` event is meant to be emitted whenever the tax of a specific pool is updated. However, in the `FundMeToken` contract, when `funding >= fundingCap`, both the `buyTax` and `sellTax` of the `pool` are reset to `0` without emitting the `TaxToken.TaxesUpdated` event. This omission can cause issues with off-chain tracking components of the protocol, leading to unexpected behavior.

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/tokens/TaxToken.sol#L34

https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/tokens/FundMeToken.sol#L41-L48

#### Recommendation
Consider modifying the `receive()` function in the `FundMeToken` contract to emit the `TaxesUpdated` event when the taxes are reset, as shown below:

```diff
receive() external payable {
+    if (funding < fundingCap) {
        funding += msg.value;
        if (funding >= fundingCap) {
            buyTax[address(pool)] = 0;
            sellTax[address(pool)] = 0;
+            emit TaxesUpdated(address(pool), 0, 0);
        }    
+    }
    
    payable(treasury).transfer(msg.value);
}
```
**Note:** This fix does not address the scenario where `fundingCap == 0` and `buyTax[address(pool)] | sellTax[address(pool)] != 0`. It is assumed that when the token is deployed with a `fundingCap` of `0`, both buy and sell taxes should already be set to `0` at deployment.

#### Discussion
Protocol team: Acknowledged

### <a id="L03"></a> [L-03] Constant Variable Naming Should Follow UPPER_CASE_WITH_UNDERSCORES Convention

#### Description
According to the [Solidity style guide](https://docs.soliditylang.org/en/v0.8.10/style-guide.html#constants):

> Constants should be named using all uppercase letters with underscores separating words (e.g., MAX_BLOCKS, TOKEN_NAME, TOKEN_TICKER, CONTRACT_VERSION).

However, the constant variable `vestingPeriod` in the `VestingLaunchToken` contract uses mixed case, which may lead to confusion when reading the code.

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/tokens/VestingLaunchToken.sol#L15

#### Recommendation
Rename the constant from `vestingPeriod` to `VESTING_PERIOD` for consistency with the recommended naming convention.

#### Discussion
Protocol team: Acknowledged

### <a id="L04"></a> [L-04] Redundant Storage Variable `dispatcher` in `FundMeToken` Contract

#### Description
The storage variable `dispatcher` in the `FundMeToken` contract is not utilized anywhere in the code, leading to unnecessary gas consumption during contract deployment.

#### Code Snippet
https://github.com/Trumpero/goat-trading-dojo-audit_2024_10_23/blob/53197145fdce52b2d8a5ff293458970d6499b735/src/tokens/FundMeToken.sol#L21

#### Recommendation
Remove the unused storage variable `dispatcher` to optimize gas usage.

#### Discussion
Protocol team: Acknowledged