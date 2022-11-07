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

A very common misconception has been that the Merge would have reduced gas fees, mainly because that was the main perceived issue over the period of its development and because the massive reduction in the energy consumption of the network was a suggestion that some cost could be cut down. And that was partially true, not for transaction fees but for block rewards. Since EIP-1559 there has been a clear distinction between the roles of block rewards and transaction fees: the former to incentivize block producers to run their software and the latter to prevent the blockchain to become too big and expensive to run on a node.

Since the Merge cut down the cost to run a block producer by a lot, issuance was reduced but with no scalability improvement and therefore no cut in tx fees. This is what EIP-4844 is meant for.

## Scalability bottlenecks
The Ethereum blockchain artificially limits its throughput to prevent the network from centralizing due to an increase of the hardware requirements to run a node. The resources that a transaction consumes are therefore carefully priced using gas, but the mechanism reaches its limits when dealing with increasing the size of the state of the history of the blockchain, since that storage gets consumed permanently but only gets paid once. State growth is even worse than history growth since it also increases the worst case time to process a transaction.


## The Blobspace

EIP-4844 introduces a new transaction type
