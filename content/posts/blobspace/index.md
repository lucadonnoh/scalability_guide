+++
title = "Deep dive into the blobspace"
date = "2022-11-07T19:16:03+01:00"
author = "donnoh.eth"
authorTwitter = "donnoh_eth" #do not include @
cover = "cover.png"
tags = ["eip-4844", "proto-danksharding", "danksharding", "blobspace", "fraud-proofs", "data-availability-sampling"]
keywords = ["", ""]
description = "EIP-4844 and Danksharding explained in depth."
showFullContent = false
readingTime = true
hideComments = false
draft = true
path = "https://scalability.guide/posts/blobspace/"
autonumbering = true
+++

That the Merge would have reduced gas fees has been a very common misconception, mainly because that was the main perceived issue over the period of its development and because the massive reduction in the energy consumption of the network was a suggestion that some cost could be cut down. And that was true, not for transaction fees but for block rewards. Since EIP-1559 there has been a clear distinction between the roles of block rewards and transaction fees: the former to incentivize block producers to run their software and the latter to prevent the blockchain to become too big and expensive to run on a node.

Since the Merge cut down the cost to run a block producer by a lot, issuance was reduced  with no scalability improvement and therefore no cut in tx fees. EIP-4844 introduces the blobspace to fix the issue.

## Scalability bottlenecks
The Ethereum blockchain artificially limits its throughput to prevent the network from centralizing due to an increase of the hardware requirements to run a node. The resources that a transaction consumes are therefore carefully priced using gas, but the mechanism reaches its limits when dealing with increasing the size of the state or the history of the blockchain, since that disk space gets consumed permanently but only gets paid once. State growth is even worse than history growth since on top of that it also increases the worst case time to process a transaction, and that's why it's much more expensive to store data in the state than in history with calldata. Full nodes are required to store all the history which occupies alone more than 400GB of disk space and is only retrieved when data is requested through RPC methods or when a new peer wants to sync.

Ethereum rollups need to post a lot of non-interactive data on the base layer to fully inherit it's security, which represents [around 90%](https://polygon.technology/blog/from-rollup-to-validium-with-polygon-avail) of the total cost of a L2 transaction. To make these transactions cheaper, we need to increase the throughput of raw data without increasing the hardware requirements to run a node.

## The (proto-)blobspace

Blocks have a maximum capacity of 30M gas, and at the cost of 16 gas per calldata byte each block can handle at maximum ~1.8 MB. While this is optimal for rollups, the actual average block size is about [80kB](https://etherscan.io/chart/blocksize) because the blockspace is contended between other resources like execution, memory and state storage.

The introduction of the blobspace creates a different space for raw data with its own EIP-1559 fee market: 16 max blobs per block with a target of 8 that are priced in gas. Each blob can be up to ~125kB, so the blobspace can handle up to 2MB of data per block with an average of 1MB. While this decoupling is great, 1MB per block is about 2.5TB per year, a far higher growth rate than Ethereum requires today.

The solution comes from the realization that **the purpose of Ethereum is not to guarantee the storage of all historical data forever**: rollups need to publish data on Ethereum to make it available long enough so others can verify it, eventually executing fault proofs, and pick it up. 