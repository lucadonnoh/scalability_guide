+++
title = "Optimism Bedrock vs Arbitrum Nitro: Deposits"
date = 2022-08-29T17:45:41+02:00
author = "donnoh.eth"
authorTwitter = "donnoh_eth" #do not include @
cover = "cover.png"
tags = ["bedrock", "nitro", "optimistic-rollup", "optimism", "arbitrum"]
keywords = ["", ""]
description = "Differences and similarities of Optimism Bedrock and Arbitrum Nitro."
showFullContent = false
readingTime = true
hideComments = false
draft = false
path = "https://scalability.guide/posts/bedrock_vs_nitro/deposits/"
autonumbering = true
+++

This was initially meant to be the first of a series of articles discussing the differences and similarities between the designs of Optimism Bedrock and Arbitrum Nitro, but norswap frontrun me with this [amazing article](https://norswap.com/bedrock-vs-nitro/), go check it out!

## Deposits

Deposited transactions are L2 transactions that are submitted using a smart contract on L1. They are mainly used for depositing assets on the rollup chain, while other L2 transactions are directly sent to the sequencer. If the sequencer is offline or censoring, this mechanism can be used to inherit L1 liveness and censorship resistance.

The main difference between the two rollups is the delay separating the submission and the inclusion of the transaction on L2: **Nitro** uses a queue of timestamped transactions where the sequencer is allowed to wait some time (around 10 mins = ~50 Ethereum blocks) before including them to prevent dealing with reorgs. Only after 24 hours the tx can be force included with a second transaction on L1: a whole day for each transaction if the sequencer is offline or censoring. The reason behind this high delay is to disincentivize forced transaction under normal conditions because they directly affect the state for any unconfirmed (not batched) L2 transaction, invalidating soft-confirmation receipts.

Deposited transactions on **Bedrock** are automatically force included in an L2 block as soon as possible: by specification, the first block in a rollup epoch must include all the deposited transactions in the corresponding L1 block (one epoch per L1 block and possibly many L2 blocks per epoch). This means that in the case of a sequencer failure the L2 chain can continue to operate without any additional artificial delay, but with deposited transactions always frontrunning unconfirmed transactions. Dealing with reorgs is not avoided.

## Address aliasing

Imagine this situation: an L2 contract holds some funds and has a function that allows another L2 contract to withdraw them. This last contract, defined by its code, can call the function only with low amounts and with a low time frequency. If this contract has been created using the `CREATE` opcode (which calculates its address as `keccak(senderAddress, nonce)`), the deployer could create a contract on L1 with the same address but a different code. This allows the attacker to use a deposited transaction to withdraw the funds from the fund holding contract with high frequency and high amounts because the sender address would be the same.

**Both rollups** prevent this type of attack by adding a constant number (they both use `0x1111000000000000000000000000000000001111`) to the sender address if it belongs to a smart contract.

## Gas market

Since a deposited transaction is submitted on L1 but executed on L2, there has to be some mechanism to pay for the gas used on L2.

One solution that **Bedrock** proposes (but does not implement yet) is a direct payment sent using the bridge. While this is ideal, it implies every caller to be marked as `payable` which is not possible for many existing projects. The alternative it implements is to pay by burning gas: a while loop burns an amount of ETH equal to $c = g \cdot b_{\text{L2}}$, where $b_{\text{L2}}$ is the basefee of the L2 and $g$ is the quantity of gas requested. The gas required to process the deposits gets credited, and if this amount is greater than the requested gas then no gas is burned. The L2 basefee cannot be known on L1, so it gets weakly estimated using a EIP-1559-like algorithm.

**Nitro** implements the direct payment solution. Users can pay by sending funds through the bridge or, if this is not possible, by funding the L2 address beforehand. It is not possible to pay directly on L1 for the gas used on L2.

## Failed transactions

Since a deposited transaction is executed on L2, the bridge cannot know whether it succeded or not until much later and funds could be lost in the meantime. A smart contract that sends a transaction through the bridge with both callvalue on L1 and L2 can lose its funds if it does not implement a way to retry the call without the L1 callvalue.

**Nitro** implements a system of "retryable tickets": if a deposited transaction fails, ArbOS creates a retryable ticket that can be retried ("redeemed") by anyone that provides funds for its gas. If the transaction has some callvalue attached, it gets escrowed by ArbOS. After 1 week an unredeemed ticket expires and gets deleted. A failed retry can be retried and an expiring ticket can be renewed. When creating a retryable, a beneficiary address can be provided to receive the escrowed funds if the ticket gets deleted. The beneficiary can also cancel the ticket. Redeems are always placed in the same block as the transaction that requested it.

**Bedrock** uses the CrossDomainMessenger wrapper around the core deposit contract to retry failed deposited transactions. A failed transaction on L2 can be replayed by calling the `relayMessage` function at any time.


## Other considerations

The **Nitro** deposit contract is pausable and implements an allow list, while the **Bedrock** deposit contract does not.