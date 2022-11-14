+++
title = "Exploring the blobspace"
date = "2022-11-07T19:16:03+01:00"
author = "donnoh.eth"
authorTwitter = "donnoh_eth" #do not include @
cover = "cover.png"
tags = ["eip-4844", "proto-danksharding", "danksharding", "blobspace", "fraud-proofs", "data-availability-sampling"]
keywords = ["", ""]
description = "EIP-4844 explained."
showFullContent = false
readingTime = true
hideComments = false
draft = false
path = "https://scalability.guide/posts/blobspace/"
autonumbering = true
+++

That the Merge would have reduced gas fees has been a very common misconception, mainly because that was the main perceived issue over the period of its development and because the massive reduction in the energy consumption of the network was a suggestion that some cost could be cut down. And that was true, not for transaction fees but for block rewards. Since EIP-1559 there has been a clear distinction between the roles of block rewards and transaction fees: the former to incentivize block producers to run their software and the latter to prevent the blockchain to become too big and expensive to run on a node.

Since the Merge cut down the cost to run a block producer by a lot, issuance was reduced but with no scalability improvement and therefore no cut in tx fees. EIP-4844 introduces the blobspace to fix the issue.

## Scalability bottlenecks

The Ethereum blockchain artificially limits its throughput to prevent the network from centralizing due to an increase of the hardware requirements to run a node. The resources that a transaction consumes are therefore carefully priced using gas, but the mechanism reaches its limits when dealing with increasing the size of the state or the history of the blockchain, since that disk space gets consumed permanently but only gets paid once. State growth is even worse than history growth since on top of that it also increases the worst case time to process a transaction, and for this reason it's much more expensive to store data in the state than in history with calldata. Historical blocks and receipts currently occupy alone more than 400GB of disk space and is only retrieved when data is requested through RPC methods or when a new peer wants to sync.

Ethereum rollups need to post a lot of non-interactive data on the base layer to fully inherit it's security, which represents [around 90%](https://polygon.technology/blog/from-rollup-to-validium-with-polygon-avail) of the total cost of a L2 transaction. To make these transactions cheaper, we need to increase the throughput of raw data while minimizing the increment in the hardware requirements to run a node.

## The (proto-)blobspace

Blocks have a maximum capacity of 30M gas, and at the cost of 16 gas per calldata byte each block can handle at maximum ~1.8 MB. While this is optimal for rollups, the actual average block size is about [80kB](https://etherscan.io/chart/blocksize) because the blockspace is contended between other resources like execution, memory and state storage.

The introduction of the blobspace creates a different space for raw data with its own EIP-1559 fee market: 16 max blobs per block with a target of 8 that are priced in gas. Each blob can be up to ~125kB, so the blobspace can handle up to 2MB of data per block with an average of 1MB. While this decoupling is great, 1MB per block is about 2.5TB per year, a far higher growth rate than Ethereum requires today.

A solution comes from the realization that **the purpose of Ethereum is not to guarantee the storage of all historical data forever**: rollups need to publish data on Ethereum to make it available long enough so others can verify it, eventually execute fault proofs, and pick it up. The current proposal aims to delete blobs after 30 days, well enough for optimistic rollups with a dispute period of 7 days, for sequencers to sync and for permanent storage solutions to archive the data.

### High-level implementation

EIP-4844 introduces a new transaction type, which we call a blob-carrying transaction. The execution layer does not have access to the blob data itself but only to a versioned hash, similar to the blockhash opcode. 

A **blob verification precompile** is added to the EVM that takes a blob hash and a blob and verifies the corrispondence. The precompile will be mainly used by optimistic rollups to provide to the execution layer the original data needed for their fault proof system.

Blob hashes are not calculated directly from their raw data but from their polynomial representation, and the intuition because this is useful will be provided later in the article. A **point evaluation precompile** takes a versioned hash, an $x$ and $y$ coordinate and a proof that $P(x)=y$ where $P$ is the polynomial represented by the blob that has the given hash. Validity rollups can be quite different from one another but they all use polynomials to represent data in their proofs. The precompile can be used to prove that the two polynomials, the one from the validity proof and the one from the blob refer to the same data.

Other changes refer to the networking of the blobspace and the block building logic to deal with blob-carrying transactions and their two-dimensional fee market.

## From proto-danksharding to full danksharding

The fact that every node needs to download and store all blobs does not scale with the size of the network, even if they are deleted after 30 days. Full danksharding is a solution to this problem based on data sampling: extend the data using Reed-Solomon encoding so only 50% is needed to be sampled to reconstruct the original data. If the attacker withholds more than 50%, the probability of sampling only available shares halves at each draw.

The Reed-Solomon encoding uses polynomials to represent and extend the data, but this alone is not enough since we have to make sure that such extension is correct without having to download all the original data (which is the whole point). One technique that has already been [discussed on this blog](https://scalability.guide/posts/maximising_light_clients_security/) is to use 2- or n-dimensional Reed-Solomon encondings, but EIP-4844 chooses KZG commitments instead because they're less complex.

By having more nodes that sample blobs it is easier to reach the 50% threshold of data sampled and therefore the network can either decrease the amount of data that each node has to sample to reduce load or increase the size of the blobspace to increase throughput.

## Concerns

During the All Core Devs Call #149 of Nov 10 2022, the Ethereum developer [Micah Zoltu](https://twitter.com/MicahZoltu) raised an important discussion about Ethereum's censhorship resistance. Micah explained that the percentage of people running a full node is 0% rounded to the nearest integer and that the introduction of the blobspace is a strict increment in operational costs, therefore turning that zero into a smallest zero and increasing the dependence of centralized and censoring services. For all the efforts that developers are pouring into increasing Ethereum's scalability through EIP-4844 there is a lack of effort to reduce the costs of running an Ethereum node.

[Dankrad Feist](https://twitter.com/dankrad) suggested that the marginal increase in operational costs would not materially damage Ethereum's censorship resistant qualities, but Micah explained that this line of reasoning can be applied to any small enough increment and that it has already been used to justify block's gas limit increments in the past. Dankrad also added that the main reason many people do not run their nodes is due to a poor user experience. [Protolambda](https://twitter.com/protolambda) proposed that making Ethereum cheaper to use would reduce the barrier to entry thus leading to more people running a node.

[Lukasz Rozmej](https://twitter.com/URozmej) questioned if EIP-4844 would actually negatively impact Ethereum's censorship resistance if it makes it harder for Ethereum validators to censor individual user transactions since many of these would be executed by L2 rollups. Micah agreed that EIP-4844 may make it harder for validators to censor individual transactions, but also pointed that at the moment there are no L2s and the only one that exists today (Fuel V1) has zero users.

Currently there are many proposal to reduce the costs of running a node:
- [Verkle trees](https://notes.ethereum.org/@vbuterin/verkle_tree_eip): reduce the witness size of accessing an account from ~3kB to ~200 bytes (using KZG commitments!)
- [State expiry](https://notes.ethereum.org/@vbuterin/state_expiry_eip): at the beginning of every period (e.g. ~1 year) a new empty state tree is initialized and any state update goes into that tree. Only the most recent tree can be modified and full nodes are expected to only hold the most recent two trees. Older objects require a witness (cheaper with Verkle trees!) to be accessed.
- [Statelessness](https://dankradfeist.de/ethereum/2021/02/14/why-stateless.html): blocks come with full witnesses and thus no state is required to validate whether a block is valid. Partial statelessness is equivalent to state expiry, while weak statelessness requires only block producers to hold the state.

## Appendix: why are polynomials so useful?

The most important property we take advantage of is that two different polynomials of degree $n$ can have at most $n$ points in common: two different lines have at most one point in common, two parabolas have at most two points in common, and so on. Evaluating a polynomial at a random point gives us a commitment to that polynomial since it's very difficult to have calculated that point with another polynomial of the same degree. KZG commitments and validity proof use this property to succintly represent data and prove knowledge of it (can be raw data, execution trace of a program or whatever). The complexity about these proof systems lies in the fact that they have to make sure that the polynomials have the same degree, that the commitment is calculated correctly, that the Prover doesn't know the random point beforehand (hidden random evaluation using homomorphic encryption) and that the system is non-interactive. These details will be better discussed in a future article.

Below is a very simple example of how polynomials can be used.

---

Alice and Bob want to determine whether the files they own are equal by minimizing the amount of information exchanged during communication. The files consist of a sequence of $n$ ASCII characters, so $m = 128$ possible values for each character. 
Alice owns the file $(a_1, \dots, a_n)$, while Bob owns the file $(b_1, \dots, b_n)$. The trivial solution is to send all $n$ characters, but this is not possible if $n$ is very large. There is no deterministic procedure that sends less information, so they decide to use a probabilistic one. One sets a prime number $p \geq \max \lbrace m,n^2 \rbrace$ for the field $\mathbb{F}_p$. For the rest of the example we assume that all operations are performed in this field. We define the family of hash functions $\mathcal{H} = \lbrace h_r : r \in \mathbb{F}_p \rbrace$ where $h_r(a_1, \dots, a_n) = \textstyle\sum_1^n a_i \cdot r^i$, i.e. the result of evaluating the polynomial of degree $n-1$ over $r$ obtained by interpreting the characters as coefficients. Alice chooses a random element $r \in \mathbb{F}_p$, computes $v = h_r(a)$ and sends $r$ and $v$ to Bob. Bob checks whether $v = h_r(b)$, if so gives `EQUAL` in output, otherwise gives `NOT EQUAL`.

### Protocol cost

The deterministic procedure has a cost of $n \log m$ bits exchanged. In the probabilistic procedure, only two elements of $\mathbb{F}$ are exchanged, namely $v$ and $r$: assuming $p \leq n^c$ for some constant $c$, the cost is $O(\log n)$.

### Completeness

Trivially, if $\forall i \in \lbrace 1, \dots ,n \rbrace : a_i = b_i$ then Bob gives in output `EQUAL` for every choice of $r$.

### Soundness

If there is at least one $i$ for which $a_i \neq b_i$, then Bob gives in output `NOT EQUAL` with probability at least $1-(n-1)/p$, that is, at least $1-1/n$ for $p \geq n^2$. To prove this, let $p_a(x)=\sum_1^n a_i \cdot x^{i-1}$ and similarly $p_b(x) = \sum_1^n b_i \cdot x^{i-1}$: if there is at least one $a_i \neq b_i$, then **there are at most $n-1$ values of $r$ such that $p_a(r) = p_b(r)$**. Since $r$ is randomly chosen by $\mathbb{F}_p$, the probability that Alice will draw $r$ is at most $(n-1)/p$, so the probability that he will output `NOT EQUAL` is at least $1-(n-1)/p$.

