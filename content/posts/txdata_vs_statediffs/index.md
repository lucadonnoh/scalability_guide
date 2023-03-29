+++
title = "Data Availability: Transaction Data vs State Diffs"
date = "2023-03-29T21:30:55+02:00"
author = "donnoh.eth"
authorTwitter = "donnoh_eth" #do not include @
cover = "cover.png"
tags = ["data-availability", "rollups"]
keywords = ["", ""]
description = "What are the trade-offs between posting transaction data and state diffs?"
showFullContent = false
readingTime = true
hideComments = false
draft = true
path = "https://scalability.guide/posts/txdata_vs_statediffs/"
autonumbering = true
+++

## What is Data Availability?

When a new block is appended to the Ethereum blockchain, a state [root](https://blog.ethereum.org/2015/11/15/merkling-in-ethereum) and a transactions root inside the block header are proposed to the network. The state root represents the output of the block's execution, in particular the state of the Ethereum Virtual Machine after each transaction in such a block has been executed.

Full nodes verify that this state root is correct by locally executing the block's transactions, therefore by calculating the new state and comparing it with the proposed state root. If a transaction is missing, the state root (or the transactions root) will be different from the one calculated by the full node, so the block will be rejected. This mechanism ensures that all the data needed to construct and verify the correctness of the blockchain is available.

Rollup bridges (in particular the L2-to-L1 side) are smart contracts that implement a trust minimized rollup light node on Ethereum. This means that such bridges do not directly execute each state transition to verify their correctness, but instead they rely on other means such as fault proofs (optimistic rollups) or validity proofs (validity rollups) to verify state roots posted by the rollup's Proposers.

