+++
title = "Forced txs vs based sequencing"
date = "2024-06-11T21:30:55+02:00"
author = "donnoh.eth"
authorTwitter = "donnoh_eth" #do not include @
cover = "cover.png"
tags = ["rollups", "sequencing"]
keywords = ["", ""]
description = "What are the tradeoffs between forced transaction mechanisms and based sequencing in rollups?"
showFullContent = false
readingTime = true
hideComments = false
draft = false
path = "https://scalability.guide/posts/forced_vs_based/"
autonumbering = true
+++

## Forced transaction mechanisms

Currently there are only two universal rollup stacks that implement a forced transaction mechanism to bypass a centralized sequencer: OP stack (e.g. OP Mainnet and Base) and Orbit stack (e.g. Arbitrum One).

### OP: deposited transactions

Users can force L1->L2 messages by calling the `depositTransaction` function on the OptimismPortal contract on L1. They are forced transactions because the derivation rule enforces their automatic inclusion in the L2 chain. OP Mainnet block time is 2 seconds, meaning that there are 6 L2 blocks for each L1 block. These sequences of L2 blocks between L1 blocks are called "epochs", therefore for each L1 block there is a corresponding epoch on L2 containing 6 blocks. The derivation rule enforces that the first portion of the first block of an epoch includes deposited transactions included in the corresponding L1 block, while the rest of the epoch contains L2 sequenced transactions.

### Orbit: Inbox and DelayedInbox

The Inbox defines the canonical chain data and the sequencer sends L2 sequenced transaction data there. L1->L2 messages are sent instead to the DelayedInbox. The sequencer can decide to move transactions from the DelayedInbox to the Inbox to include them in the chain, but if it doesn't do so within a certain time frame, users can move the transactions themselves.

### Sequencer failures

Since the sequencer is centralized, it can fail to post batches onchain but it can continue to provide preconfirmations to users not to halt the chain. These scenarios happen more often than it is discussed, sometimes due to bugs as in the recent Degen chain incident, other times due to scheduled node upgrades.

{{< figure src="base.png" alt="Base downtime" position="center" style="border-radius: 8px;" caption="Base halting batch posting for 1h a few days ago. I bet no one even noticed." >}}

If the Base sequencer was not able to backdate transactions, L2 blocks between L1 blocks would only contain deposited transactions and the rest would have been empty, meaning that it would not have been possible to provide real time preconfirmations. If the Base sequencer can backdate transactions from the time of posting on L1, one cannot derive the state that includes the deposited transactions because it is not known if the sequencer will include L2 sequenced transactions before them. There is a limit to how much the sequencer can backdate transactions, which is currently set to 12 hours on OP stack chains. This means that forced transaction derivation can be delayed up to 12 hours, which implies that no state roots can be posted during that time and therefore withdrawals cannot be processed. If the sequencer remains offline for more than 12 hours, the chain gets filled with empty blocks and deposited transactions gets processed.

The Arbitrum sequencer can also backdate transactions but only limited to a few minutes to account for L1 reorgs. The DelayedInbox takes care of sequencer downtime by allowing users to move transactions to the Inbox permissionlessly only after a certain time frame, currently 24 hours. In the same way, it implies that forced withdrawals cannot be processed during that time.

### Cost of forcing via L1

[Here](https://etherscan.io/tx/0x0a1695ac95217f245249dfc72419a67738eba4ac708b1a722f28814970200f49) an example of a deposited transaction on OP Mainnet Optimism portal. It spent 60'702 gas, which at 20 gwei is around \$4. In Arbitrum one, a transaction to the DelayedInbox ([here](https://etherscan.io/tx/0x87fa3c2265f75eb9f1c79fea81865a41f14b428b90811224a7ab32c6799dceb1)) spends 91'261 gas, which is around \$6 at 20 gwei. In comparison, the average cost of a L2 sequenced transaction is respectively around \\$0.0007 and \\$0.0021.

## Based sequencing

If a rollup decides to use based sequencing there is no centralized sequencer that needs a bypass mechanism. The rollup inherits by default the liveness and censorship resistance properties of L1. Moreover, there is no need to account for failures since L1 is usually live. The problem arises with providing preconfirmations that are faster than L1 inclusion confirmations, i.e. 12 seconds. Taiko is the only based rollup live in production and its current UX suffers from this issue. While the details of based preconfirmations are still being discussed, based preconfers are expected to have a stake that can economically guarantee the correctness of the preconfirmation since it can be slashed if they equivocate, providing net benefits compared to fully trusted preconfirmations. With based sequencing users can exit without incurring in any additional cost or delay. Rollup teams can also obtain plausible deniability in case of sequencing of illegal transactions.
