# Goat Trading Security Review - 2024/06/05
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
The [goat-trading](https://github.com/inedibleX/goat-trading) repository was audited at commit [0b847c1e66410f7ec9f2317766cc639049217195](https://github.com/inedibleX/goat-trading/commit/0b847c1e66410f7ec9f2317766cc639049217195) 

The following contracts were in scope:
* contracts/tokens/LotteryToken.sol 
* contracts/tokens/LotteryTokenMaster.sol 


# Findings Summary 
| ID           | Title                                                                                                                                                        | Severity | Status  |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |  -------- | ------- |
| [C-01](#C01) | Missing update of `entryIndex` before returning in the `upkeep` function | CRITICAL | Fixed |
| [C-02](#C02) | The `upkeep` function still checks for wins for entries with drawBlock equal to `block.number`, leading to randomization of the lottery is broken | CRITICAL | Fixed |
| [H-01](#H01) | Incorrect condition for the loop index in the `upkeep` function. | HIGH | Fixed |
| [L-01](#L01) | The `createLotteryToken` function should validate that `initParams.initialEth` is not zero | LOW | Acknowledged |
| [L-02](#L02) | Variable `lotteryMaster` can be set to `immutable` | LOW | Acknowledged |
| [L-03](#L03) | Redundant check whether `lotteryMaster` is different from `address(0)` | LOW | Acknowledged |

# Detailed Findings

## <a id="Critical"></a>Critical

### <a id="C01"></a> [C-01] Missing update of `entryIndex` before returning in the `upkeep` function
#### Description
The `upkeep` function is used to loop through entries to check for winners. This function uses a storage variable `entryIndex` to be the start index of the loop, which is updated to the index of the last checked entry after the loop ends.

However, when `entry.drawBlock > block.number`, this function will return without updating that variable. In that case, even though the entries were looped through and checked, the next call of `upkeep` will still use the stale entryIndex, causing it to loop through the previously checked entries again.
```solidity=
for (uint256 i = startIndex; i < _loops && i < entriesLength; i++) {
    Entry memory entry = entries[i];

    // Must have reached draw block. If not, end of checks.
    if (entry.drawBlock > block.number) return;

   ...
}
if ((startIndex + _loops) < entriesLength) {
    entryIndex += _loops;
} else {
    entryIndex = entriesLength;
}
```
#### Impact
The entries may be repeatedly considered in the `upkeep` function because `entryIndex` wasn't updated to the index of the latest entry. This leads to the risk of repeatedly distributing rewards to the same entry, while the remaining entries may not be processed.
#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/lottery-token/contracts/tokens/LotteryTokenMaster.sol#L121
#### Recommendation
Update `entryIndex` before returning in the `upkeep` function:
```solidity=
if (entry.drawBlock > block.number) {
  entryIndex = i;
  return;
}
```


#### Discussion
Protocol team: fixed 

### <a id="C02"></a> [C-02] The `upkeep` function still checks for wins for entries with drawBlock equal to `block.number`, leading to randomization of the lottery is broken
#### Description
In the LotteryTokenMaster contract, the `upkeep` function is used to loop through entries to check for wins. It limits the draw block to the current `block.number` with the following condition:
```solidity=
if (entry.drawBlock > block.number) return;
```
It means this function still checks for a win for entries where the draw block is equal to the current `block.number`. 
However, it uses the block hash of the draw block to check the win probability of the entry. When the draw block is equal to the current `block.number`, this block is still incomplete, so the block hash will be 0. This breaks the randomization of the lottery because anyone can prepare an address to win with a 0 block hash and call `upkeep` until their draw block is equal to `block.number`, ensuring they always win.

#### Impact
An attacker can use an address that will always win with a block hash of 0 to create an entry and call `upkeep` until their draw block is equal to `block.number`. By doing this, the attacker will always win and claim all the rewards of the lottery token.
#### Code snippet
https://github.com/inedibleX/goat-trading/blob/lottery-token/contracts/tokens/LotteryTokenMaster.sol#L121
#### Recommendation
The condition for returning should be `entry.drawBlock >= block.number` instead of `entry.drawBlock > block.number`
#### Discussion
Protocol team: fixed  

---

## <a id="High"></a>High

### <a id="H01"></a> [H-01] Incorrect condition for the loop index in the `upkeep` function.
#### Description
In the `upkeep` function of the LotteryTokenMaster contract, `_loops` is the variable that represents the number of entries to check. However, in the loop condition, it uses `i < _loops` to limit the index i. This is incorrect because the loop starts from startIndex, so the condition for i should be `i < startIndex + _loops`.
```solidity=
function upkeep(uint256 _loops) external {
  // Tokens will call with 0 so we can adjust default as needed.
  // Keepers can call with many more if necessary.
  if (_loops == 0) _loops = defaultUpkeepLoops;
  uint256 startIndex = entryIndex;
  uint256 entriesLength = entries.length;
  
  for (uint256 i = startIndex; i < _loops && i < entriesLength; i++) {
      ...
 ```
#### Impact
The number of loops in the upkeep function will be incorrect
#### Code snippet 
https://github.com/inedibleX/goat-trading/blob/lottery-token/contracts/tokens/LotteryTokenMaster.sol#L117
#### Recommendation
The condition `i < _loops` should be replaced with `i < startIndex + _loops`

#### Discussion
Protocol team: fixed 

---

## <a id="Low"></a>Low 

### <a id="L01"></a> [L-01] The `createLotteryToken` function should validate that `initParams.initialEth` is not zero
#### Description
The `createLotteryToken` function attempts to create a pair with `initParams`. However, this function doesn't support `initParams.initialEth` > 0 since it is not a payable function and doesn't transfer ETH to the pool. Therefore, `initParams.initialEth` should be validated to ensure it is not zero at the beginning of the function.
#### Code snippet
https://github.com/inedibleX/goat-trading/blob/lottery-token/contracts/tokens/LotteryTokenMaster.sol#L85-L88
#### Recommendation
This check should be added at the beginning of the `createLotteryToken` function:
```solidity=
if (initParams.initialEth != 0) {
    revert TokenErrors.InitialEthNotAccepted();
}
```
#### Discussion
Protocol team: acknowledged

### <a id="L02"></a> [L-02] Variable `lotteryMaster` can be set to `immutable`
#### Description
The `lotteryMaster` variable is initialized in the `LotteryToken` constructor and cannot be modified afterwards. Therefore, we can declare this variable as `immutable` to save gas.

#### Code snippet
https://github.com/inedibleX/goat-trading/blob/75d20224567a0004d34b176dddff9cbcd33e8039/contracts/tokens/LotteryToken.sol#L26

#### Recommendation
Modify the `lotteryMaster` variable to `immutable`.
#### Discussion
Protocol team: acknowledged

### <a id="L03"></a> [L-03] Redundant check whether `lotteryMaster` is different from `address(0)`
#### Description
The `lotteryMaster` variable is initialized to `msg.sender` in the `LotteryToken` contract's constructor. Since `msg.sender` can never be `address(0)`, the value of `lotteryMaster` will also never be `address(0)`. Additionally, there is no setter function to change the value of `lotteryMaster`, ensuring it is always non-zero.

Therefore, any checks in the `LotteryToken` contract to verify that `lotteryMaster` is different from `address(0)` are redundant.

#### Code snippet
* https://github.com/inedibleX/goat-trading/blob/75d20224567a0004d34b176dddff9cbcd33e8039/contracts/tokens/LotteryToken.sol#L51
* https://github.com/inedibleX/goat-trading/blob/75d20224567a0004d34b176dddff9cbcd33e8039/contracts/tokens/LotteryToken.sol#L78

#### Recommendation
Consider removing all checks that `lotteryMaster != address(0)` to save gas.
#### Discussion
Protocol team: acknowledged
