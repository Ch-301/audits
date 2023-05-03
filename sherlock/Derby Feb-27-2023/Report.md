Audit Report - Derby


##Summary
table here

##Findings

High  Risk Findings (n)

Medium Risk Findings (n)

[M-01] Vault could rebalance() before funds arrive from xChainController

Summary
Invoke sendFundsToVault() to Push funds from xChainController to vaults. which is call xTransferToVaults()

For the cross-chain rebalancing xTransferToVaults() will execute this logic

       ...
      pushFeedbackToVault(_chainId, _vault, _relayerFee);
      xTransfer(_asset, _amount, _vault, _chainId, _slippage, _relayerFee);
       ...
pushFeedbackToVault() Is to invoke receiveFunds()
pushFeedbackToVault() always travel through the slow path
xTransfer() to transfer funds from one chain to another
If fast liquidity is not available, the xTransfer() will go through the slow path.
The vulnerability is if the xcall() of pushFeedbackToVault() excited successfully before xTransfer() transfer the funds to the vault, anyone can invoke rebalance() this will lead to rebalancing Vaults with Imperfect funds (this could be true only if funds that are expected to be received from XChainController are greater than reservedFunds and liquidityPerc together )

Vulnerability Detail
The above scenario could be done in two possible cases
1- xTransfer() will go through the slow path but because High Slippage the cross-chain message will wait until slippage conditions improve (relayers will continuously re-attempt the transfer execution).

2- Connext Team says

All messages are added to a Merkle root which is sent across chains every 30 mins
And then those messages are executed by off-chain actors called routers

so it is indeed possible that messages are received out of order (and potentially with increased latency in between due to batch times) 
For "fast path" (unauthenticated) messages, latency is not a concern, but ordering may still be (this is an artifact of the chain itself too btw)
one thing you can do is add a nonce to your messages so that you can yourself order them at destination
so pushFeedbackToVault() and xTransfer() could be added to a different Merkle root and this will lead to executing receiveFunds() before funds arrive.

Impact
The vault could rebalance() before funds arrive from xChainController, this will reduce rewards

Code Snippet
Tool used
Manual Review

Recommendation
Check if funds are arrived or not