
# Audit Report - Blueberry

|             |                                                                           |
| ----------- | ------------------------------------------------------------------------- |
| **Date**    | July 2023                                                             |
| **Auditor** | Chouaib Bahri (Ch301) ([@0xch301](https://twitter.com/0xch301)) |
| **Status**  | **Final**                                                                 |

# Scope

The review focused on the commit hash [`007a0f16e878d46d116fdeef33f2bd6ae1db16ec`](https://github.com/Blueberryfi/blueberry-core/tree/007a0f16e878d46d116fdeef33f2bd6ae1db16ec) of the Private https://github.com/Blueberryfi/blueberry-core GitHub repository.

However I had only three days. I was only able to deep dive into `AuraSpell.sol` + `wAuraPools.sol` and especially `openPositionFarm()`

# About **Blueberry**
Blueberry unifies the DeFi experience: Aggregating, Automating, and Boosting Capital Efficiency for top DeFi Strategies.

---
# Severity Structure

The vulnerability severity is calculated based on two components: 
- Impact of the vulnerability.
- Probability of the vulnerability.

| Severity               | **Probability: Rare** | **Probability: Unlikely** | **Probability: likely** |**Probability: Very likely**|
| ---------------------- | --------------------- | ------------------------- | ----------------------- | -------------------------- |
| **Impact: Low/Info**   | Low/Info              | Low/Info                  | Medium                  |Medium                      |
| **Impact: Medium**     | Low/Info              | Medium                    | Medium                  |High                        |
| **Impact: High**       | Medium                | Medium                    | High                    |Critical                    |
| **Impact: Critical**   | Medium                | High                      | Critical                |Critical                    |

# Findings Summary

| ID     | Title                                                                                       | Severity |
| ------ | ------------------------------------------------------------------------------------------- | -------- |
| [C-01] | Front-run attack, user will lose their BPT                                                  | Critical |
| [C-02] | Users will receive more or fewer rewards than they are worth                                | Critical |
| [H-01] | Users will lose some of their rewards favor to attacker                                     | High     |
| [H-02] | The logic of `AuraSpell` is not compatible with all the AURA POOLS                          | High     |
| [H-03] | Users could lose some AURA rewards                                                          | High     |
| [H-04] | Malicious users could keep forcing transitions to revert                                    | High     |

---
# Detailed Findings

# [C-01] Front-run attack, user will lose their BPT

## Severity

**Impact:**
Critical

**Probability:**
Very likely

## Context:

[AuraSpell.sol](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/spell/AuraSpell.sol#L124)

## Description

In this line 
```solidity
userData: abi.encode(1, amountsIn, 0),
```

This is how the `userData` looks like
```
[EXACT_TOKENS_IN_FOR_BPT_OUT, amountsIn, minimumBPT]
```
So the zero value here is for `minimumBPT`

I highly recommend you don't use this: `minimumBPT == 0`

This is you telling the pool that you're happy to put in your tokens and get no BPT in return.
Someone could sandwich this transaction and totally screw you over 

Now the impact:
In this code, if a user sets 3x leverage an Attacker (MEV bot) could take 2/3 from the BPT this will lead the next check `_validateMaxLTV()` to be bypassed

## Recommendations
Set the `minimumBPT` to a reasonable value, rathert than using a zero value

You can allow much less slippage to avoid ripping off your users.
Alos, Don't use a simple percentage.

# [C-02] Users will receive more or fewer rewards than they are worth 

## Severity

**Impact:**
High

**Probability:**
Very likely

## Context:

[WAuraPools.sol](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/wrapper/WAuraPools.sol#L242-L259)

## Description

This FOR loop is for the additional rewards (Extra Rewards)

```solidity
        for (uint256 i; i != extraRewardsCount; ) {
            address rewarder = extraRewards[i];
            uint256 stRewardPerShare = accExtPerShare[tokenId][
                extraRewardsIdx[rewarder] - 1
            ];
            tokens[i + 2] = IAuraRewarder(rewarder).rewardToken();
            rewards[i + 2] = _getPendingReward(
                stRewardPerShare,
                rewarder,
                amount,
                lpDecimals
            );

            unchecked {
                ++i;
            }
        }
    }
```
`extraRewardsIdx[rewarder] - 1` this index is for this array [extraRewards[ ]](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/wrapper/WAuraPools.sol#L54)

and we can see that clearly in [_syncExtraReward](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/wrapper/WAuraPools.sol#L377-L382)

 It could be the same with `accExtPerShare[ ]` if you are interacting only with one pool but this is not the case

Now impact:
You could send more or less rewards to the users

## Recommendations
Update the logic


# [H-01] Users will lose some of their rewards favor to attacker

## Severity

**Impact:**
High

**Probability:**
Unlikely

## Context

[AuraSpell.sol](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/spell/AuraSpell.sol#L144-L145)

## Description

In case the contract of [auraRewarder] (https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/wrapper/WAuraPools.sol#L320) was updated with additional new tokens in their `extraRewards[ ]` (or even it didn't have any `extraRewards` at the beginning).

In this [part](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/wrapper/WAuraPools.sol#L332-L341) `_syncExtraReward()` function will be triggered to add the new tokens to `extraRewards[ ]`, so users could be qualified to receive rewards in these tokens

Now the impact:
In the step 6 `Take out existing collateral and burn`

```solidity
File: AuraSpell.sol

        // 6. Take out existing collateral and burn
        IBank.Position memory pos = bank.getCurrentPositionInfo();
        if (pos.collateralSize > 0) {
            (uint256 pid, ) = wAuraPools.decodeId(pos.collId);
            if (param.farmingPoolId != pid)
                revert Errors.INCORRECT_PID(param.farmingPoolId);
            if (pos.collToken != address(wAuraPools))
                revert Errors.INCORRECT_COLTOKEN(pos.collToken);
            (address[] memory rewardTokens, ) = IERC20Wrapper(pos.collToken)
                .pendingRewards(pos.collId, pos.collateralSize);
            bank.takeCollateral(pos.collateralSize);
            wAuraPools.burn(pos.collId, pos.collateralSize);
            // distribute multiple rewards to users
            uint256 rewardTokensLength = rewardTokens.length;
            for (uint256 i; i != rewardTokensLength; ) {
                _doRefundRewards(
                    rewardTokens[i] == STASH_AURA ? AURA : rewardTokens[i]
                );
                unchecked {
                    ++i;
                }
            }
        }
```

by invoking `pendingRewards()` befor `burn()` function You are excluding the user from receiving his awards 
because `rewardTokens[ ]` array isn't complete with all the `extraRewards` address.

This will allow any attacker to withdraw these rewards so the users lossing funds (Rewards)

## Recommendations
You need to postpone this line until the sub-call of `burn()`


# [H-02] The logic of `AuraSpell` is not compatible with all the AURA POOLS

## Severity

**Impact:**
Medium

**Probability:**
Very likely

## Context:

[AuraSpell.sol](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/spell/AuraSpell.sol#L298-L302)

## Description

In this Part
```solidity
        uint256 totalLPSupply = IBalancerPool(lpToken).totalSupply();
        // compute in reverse order of how Balancer's `joinPool` computes tokenAmountIn
        uint256 poolAmountOut;
        for (i = 0; i != length; ) {
            if ((maxAmountsIn[i] * totalLPSupply) / balances[i] != 0) {
                poolAmountOut = type(uint256).max;
                break;
            }

            unchecked {
                ++i;
            }
        }
```
will be not usful in case [preminted BPT](https://docs.balancer.fi/concepts/advanced/preminted-bpt.html)
Also [this](https://docs.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html#informational) Note
```
Don't use `totalSupply` without reading this first!

Balancer Pools with pre-minted BPT will always return type(uint211).max.
```
Or you can check the followin comment [HERE](https://github.com/balancer/balancer-v2-monorepo/blob/master/pkg/pool-stable/contracts/ComposableStablePool.sol#L42-L52)

Now the impact:
this multiplication `(maxAmountsIn[i] * totalLPSupply) / balances[i]` will revert due to `overflow error`. Because the `totalLPSupply == 2**(111)`

## Recommendations
You can use somthing like this
`if ((totalLPSupply == type(uint211).max) || (maxAmountsIn[i] * totalLPSupply) / balances[i] != 0)`

or just use `getVirtualSupply()` in case the preminted BPT


# [H-03] Users could lose some AURA rewards

## Severity

**Impact:**
High

**Probability:**
likely

## Context:

[WAuraPools.sol](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/wrapper/WAuraPools.sol#L164-L204)

## Description

The logic of `_getAuraPendingReward()` is not for the on-chain, as AuraMinter after can mint additional tokens after `inflationProtectionTime` has passed, those new tokens are not taken into consideration in this library.
check [this](https://vscode.blockscan.com/ethereum/0x744Be650cea753de1e69BF6BAd3c98490A855f52) and [this](https://docs.aura.finance/developers/how-to-___/see-reward-tokens-yield-on-aura-pools#dont-use-onchain-queries) 

## Recommendations
If you accept this risk you should add a comment here. 
Add a section like this "Liquidity Providers in this pool face the following risks:" in your docs. 

# [H-04] Malicious users could keep forcing transitions to revert

## Severity

**Impact:**
High

**Probability:**
likely

## Context:

[AuraSpell.sol](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/spell/AuraSpell.sol#L106)

## Description

Malicious users could keep sending dust amounts of tokens to Aura Spell. 
and [_getJoinPoolParams](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/spell/AuraSpell.sol#L282) will set these amounts to send them to `Balancer_Vault`,
But since the tokens are not `Approved` the transaction will revert   


## Recommendations
You can add an `Approve` [here](https://github.com/Blueberryfi/blueberry-core/blob/007a0f16e878d46d116fdeef33f2bd6ae1db16ec/contracts/spell/AuraSpell.sol#L282) 
Or just set them (the wrong token) to zero 

