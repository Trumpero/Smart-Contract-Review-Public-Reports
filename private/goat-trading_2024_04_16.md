# Goat Trading Security Review - 2024/04/16
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
https://github.com/inedibleX/goat-trading/tree/f60757d19fcc98e21bb075e9c790b473ce5a5326

![Screenshot 2024-05-05 at 09 30 28](https://gist.github.com/assets/44105108/3d1f659f-f0ae-48e8-bc43-7fd113ed86fa)

# Findings Summary 
| ID           | Title                                                                                                                                                        | Severity | Status  |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |  -------- | ------- |
| [C-01](#C01) | Missing subtraction of the `vaultEth` when redeeming in the VaultToken contract | CRITICAL | Fixed |
| [H-01](#H01) | Missing update `rewardPerTokenStored` in the function `addDividend` of DividendToken contract | HIGH | Fixed |
| [H-02](#H02) | The TaxShareToken contract should exclude the balances of its address and taxed addresses from reward distribution | HIGH | Fixed |
| [M-01](#M01) | TaxToken should also sell taxes when the collected tax value is lower than `minSell` | MEDIUM | Fixed |
| [M-02](#M02) | TaxToken doesn't revoke approval of the old DEX when changing to the new DEX | MEDIUM     | Fixed |
| [M-03](#M03) | Mismatch between ethValue and the actual received Eth when selling taxes | MEDIUM | Fixed |
| [M-04](#M04) | Risk of using `transfer` to send Eth | MEDIUM     | Acknowledged |
| [L-01](#L01) | The underscore storage variable should be internal | LOW     | Fixed |
| [L-02](#L02) | The function `createToken()` includes a redundant payable keyword | LOW   | Fixed |

# Detailed Findings

## <a id="Critical"></a>Critical

### <a id="C01"></a> [C-01] Missing subtraction of the `vaultEth` when redeeming in the VaultToken contract
#### Description
VaultToken is intended to work as a vault, where an amount of this token can be used to claim Eth from the contract. Eth will be deposited via the `deposit()` function:
```solidity=
function deposit() external payable {
    vaultEth += msg.value;
}
```
However, the `redeem` function doesn't subtract vaultEth, resulting in a mismatch between vaultEth and the actual Eth balance of the contract.
```solidity=
function redeem(uint256 _amount) external {
    uint256 ethOwed = _amount * vaultEth / totalSupply();
    _burn(msg.sender, _amount);
    payable(msg.sender).transfer(ethOwed);
}
```
`vaultEth` will never decrease, resulting in the Eth received from redemption being inflated. It will deplete the Eth reserves of the contract quickly.

For example, if the protocol team deposits 10 Eth to VaultToken, and there are only 2 holders: Alice and Bob, each with 50% of the tokens. When Alice redeems all of her tokens, Alice receives 5 Eth but `vaultEth` remains unchanged. After that, Bob's balance equals the total supply, so he can receive all of the vaultEth from redemption (10 Eth), which exceeds the remaining Eth in the contract (5 Eth).
#### Impact
The redemption will be inflated over time, consuming the Eth balance of the contract quickly. Consequently, the redemption of tokens will be disrupted due to the depletion of Eth.
#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/VaultToken.sol#L40-L44
#### Recommendation
Should subtract `vaultEth` by `ethOwed` in `redeem` function:
```solidity=
function redeem(uint256 _amount) external {
    uint256 ethOwed = _amount * vaultEth / totalSupply();
    _burn(msg.sender, _amount);
    payable(msg.sender).transfer(ethOwed);
    vaultEth -= ethOwed;
}
```
#### Discussion
Protocol team: fixed

---

## <a id="High"></a>High

### <a id="H01"></a> [H-01] Missing update `rewardPerTokenStored` in the function `addDividend` of DividendToken contract 
#### Description
The function `addDividend` of the DividendToken contract is used to add new rewards for the holders. This function attempts to drip total of the new rewards and remaining rewards over a targeted duration.
```solidity=
if (block.timestamp >= periodFinish) {
    rewardRate = reward / _dripTimeInSeconds;
} else {
    uint256 remaining = periodFinish - block.timestamp;
    uint256 leftover = remaining * rewardRate;
    rewardRate = (reward + leftover) / _dripTimeInSeconds;
}

// Full Ether balance.
uint256 balance = address(this).balance;
if (balance / _dripTimeInSeconds < rewardRate) {
    revert TokenErrors.ProvidedRewardsTooHigh();
}

lastUpdateTime = block.timestamp;
periodFinish = block.timestamp + _dripTimeInSeconds;
emit RewardAdded(reward);
```
However, this function misses accruing the `rewardPerTokenStored` variable during the period from the last update time to the current. In this period, rewards were still dripped by the old `rewardRate`, so the missing update will cause all active holders to lose the rewards for this time. 
In the code snippet, `lastUpdateTime` is also updated to `block.timestamp`, so the next accrued distribution will start after that. Therefore, this function must collect the last dripped rewards.

#### Impact
Token holders may lose their rewards when `addDividend` is called to add new rewards but `rewardPerTokenStored` is not updated.
#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/DividendToken.sol#L168-L183
#### Recommendation
Should accrue the dripped rewards by updating the `rewardPerTokenStored` variable in function `addDividend`:
```solidity=
rewardPerTokenStored = _rewardPerToken();

if (block.timestamp >= periodFinish) {
    rewardRate = reward / _dripTimeInSeconds;
} else {
    uint256 remaining = periodFinish - block.timestamp;
    uint256 leftover = remaining * rewardRate;
    rewardRate = (reward + leftover) / _dripTimeInSeconds;
}
```

#### Discussion
Protocol team: fixed 


### <a id="H02"></a> [H-02] The TaxShareToken contract should exclude the balances of its address and taxed addresses from reward distribution
#### Description
In the TaxShareToken contract, the `_awardTaxes` function uses the entire supply of this token to accumulate r`ewardPerTokenStored`. This means the rewards from taxes will be distributed among all holders of this token.


```solidity=
function _awardTaxes(uint256 _amount) internal override {
      uint256 reward = _amount * sharePercent / _DIVISOR;
      // 1e18 is removed because rewardPerToken is in full tokens
      rewardPerTokenStored += reward / (_totalSupply / 1e18);
      _balances[address(this)] += _amount - reward;
  }
```
However, not all addresses, including this contract's address and `taxed` addresses, will receive the reward of TaxShareToken. This is because the `_updateRewards` function does not account for these kinds of addresses, so they will not receive any rewards.
```solidity=
function _updateRewards(address _from, address _to) internal {
    if (_from != address(0) && _from != address(this) && !taxed[_from]) {
        _balances[_from] += _earned(_from);
        userRewardPerTokenPaid[_from] = rewardPerTokenStored;
    }
    if (_to != address(0) && _to != address(this) && !taxed[_to]) {
        _balances[_to] += _earned(_to);
        userRewardPerTokenPaid[_to] = rewardPerTokenStored;
    }
}
```
As a result, a portion of the reward will not be distributed to the active holders who are not taxed addresses.
#### Impact
Rewards for active holders will be lower than expected.
#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/TaxShareToken.sol#L137
#### Recommendation
Should exclude the balances of its address and taxed addresses from reward distribution in `_awardTaxes` function. Maybe you can use a storage variable to store the total balance of taxed address.

#### Discussion
Protocol team: fixed / acknowledged


---

## <a id="Medium"></a>Medium

### <a id="M01"></a> [M-01] TaxToken should also sell taxes when the collected tax value is lower than `minSell`
#### Description
The `_sellTaxes()` function of the TaxToken contract only sells an amount of tokens that have a value of Eth equivalent to `minSell` Eth each time. It's intended to swap only a small amount to avoid dumping too many tokens.
```solidity=
try IRouter(dex).getAmountsOut(tokens, path) returns (uint256[] memory amounts) {
    ethValue = amounts[1];
    if (ethValue > _minSell) {
        // In a case such as a lot of taxes being gained during bootstrapping, we don't want to immediately dump all tokens.
        tokens = tokens * _minSell / ethValue;
        // Try/catch because during bootstrapping selling won't be allowed.
        try IRouter(dex).swapExactTokensForWethSupportingFeeOnTransferTokens(
            tokens, 0, address(this), treasury, block.timestamp
        ) {} catch (bytes memory) {}
    }
} catch (bytes memory) {}
```
However, it only sells tax tokens when `ethValue > _minSell`. The problem is that `ethValue` is obtained from `IRouter(dex).getAmountsOut`, which is not always correct. The price of the DEX may be manipulated by front-running to obtain a low ethValue, allowing the accumulation of a large amount of tax tokens that are not swapped
#### Impact
The TaxToken contract carries the risk that the DEX price can be manipulated by front-running to accumulate a large number of tax tokens. Subsequently, an attacker can manipulate the significant swap to profit.
#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/TaxToken.sol#L152-L159
#### Recommendation
`_sellTaxes()` should always sell taxes, even when the collected taxes are small.


#### Discussion
Protocol team: fixed 

### <a id="M02"></a> [M-02] TaxToken doesn't revoke approval of the old DEX when changing to the new DEX
#### Description
In the TaxToken contract, the `changeDex` function approves its tokens for the new dex. However, it doesn't revoke the old approval for the old dex.
```solidity=
function changeDex(address _dexAddress) external onlyOwnerOrTreasury {
    dex = _dexAddress;
    IERC20(address(this)).approve(_dexAddress, type(uint256).max);
}
```
It causes the old DEX to still have an allowance of tokens in the contract.
#### Impact
It poses a risk that the funds of the contract will be vulnerable if the old DEX is unsafe.
#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/TaxToken.sol#L199-L202
#### Recommendation
Should revoke the allowance of the old dex: 
```solidity=
function changeDex(address _dexAddress) external onlyOwnerOrTreasury {
    IERC20(address(this)).approve(dex, 0);
    dex = _dexAddress;
    IERC20(address(this)).approve(_dexAddress, type(uint256).max);
}
```

#### Discussion
Protocol team: fixed 

### <a id="M03"></a> [M-03] Mismatch between ethValue and the actual received Eth when selling taxes
#### Description
In `TaxToken._sellTaxes()`, the ethValue variable is used to represent the received Eth when swapping taxes. It is obtained from `IRouter(dex).getAmountsOut(tokens, path)`.
```solidity=
try IRouter(dex).getAmountsOut(tokens, path) returns (uint256[] memory amounts) {
    ethValue = amounts[1];
    ...
}
```
However, the token TaxToken is a kind of fee-on-transfer token, which charges taxes for every transfer. Therefore, the actual swap amount is lower than `tokens`, so `ethValue` is higher than the actual received Eth of the swap.


#### Impact
Inconsistent behavior due to the difference between the expected tokens and the actual received tokens.
#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/TaxToken.sol#L150
#### Recommendation
Should subtract tax from `tokens` in parameter of `getAmountOut`:
```solidity=
uint256 tax = _determineTax(address(this), address(dex), tokens);
try IRouter(dex).getAmountsOut(tokens - tax, path) returns (uint256[] memory amounts) {
...
```
#### Discussion
Protocol team: fixed 

### <a id="M04"></a> [M-04] Risk of using `transfer` to send Eth
#### Description
There are multiple instances in the code that send Eth using the transfer() function.
Sending Eth with the transfer function forwards a fixed amount of gas: 2300.
If the receiver is a smart contract and it utilizes more than 2300 gas in the fallback, it will revert. Therefore, it should use call instead of transfer to support smart contract receivers.

[Consensys Blog: Stop Using Solidity's transfer Now!](https://consensys.io/diligence/blog/2019/09/stop-using-soliditys-transfer-now/ )

#### Impact
Sending Eth may revert in cases where the receiver is a smart contract.

#### Code Snippet
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/DividendToken.sol#L53
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/DividendToken.sol#L204
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/VaultToken.sol#L43
https://github.com/inedibleX/goat-trading/blob/tokens/contracts/tokens/VaultToken.sol#L82
#### Recommendation
Should use `call` instead of `transfer` to send Eth
#### Discussion
Protocol team: acknowledged 

---

## <a id="Low"></a>Low 

### <a id="L01"></a> [L-01] The underscore storage variable should be internal
#### Description
The contract `ERC20.sol` features public storage variables `_balances` and `_totalSupply`. However, these variables follow the convention of using underscores for private or internal variables in Solidity.

#### Code Snippet
- https://github.com/inedibleX/goat-trading/blob/f60757d19fcc98e21bb075e9c790b473ce5a5326/contracts/tokens/ERC20.sol#L38
- https://github.com/inedibleX/goat-trading/blob/f60757d19fcc98e21bb075e9c790b473ce5a5326/contracts/tokens/ERC20.sol#L42

#### Recommendation
It is recommended to designate the `_balances` and `_totalSupply` variables as internal.
#### Discussion
Protocol team: fixed 

### <a id="L02"></a> [L-02] The function `createToken()` includes a redundant payable keyword
#### Description
The `createToken()` function of the factory contract is marked as payable, indicating that it can receive ETH. However, the function does not utilize ETH in any actions, making the `payable` keyword redundant for this function.

#### Code Snippet
- https://github.com/inedibleX/goat-trading/blob/f60757d19fcc98e21bb075e9c790b473ce5a5326/contracts/tokens/TokenFactory.sol#L85
- https://github.com/inedibleX/goat-trading/blob/f60757d19fcc98e21bb075e9c790b473ce5a5326/contracts/tokens/TokenFactory2.sol#L74
- https://github.com/inedibleX/goat-trading/blob/f60757d19fcc98e21bb075e9c790b473ce5a5326/contracts/tokens/TokenFactory3.sol#L75

#### Recommendation
Remove the `payable` keyword from the `createToken()` function.
#### Discussion
Protocol team: fixed 

