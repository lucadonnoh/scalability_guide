+++
title = "How to build the safest (and dumbest) Optimium"
date = "2024-06-28T11:30:55+02:00"
author = "donnoh.eth"
authorTwitter = "donnoh_eth" #do not include @
cover = "cover.png"
tags = ["rollups", "optimiums"]
keywords = ["", ""]
description = "An exercise to test your mental models on rollups and optimiums."
showFullContent = false
readingTime = true
hideComments = false
draft = false
path = "https://scalability.guide/posts/safest_optimium/"
autonumbering = true
+++

## What is an Optimium?

An Optimium is a project whose bridge inherits all the security guarantees of the base layer except data availability, which is delegated to an external network that is supposed to relay "data availability attestations" to a data availability bridge, that is then referenced by the project's (optimistic) proof system. Examples of this are Celestia relaying network consensus signatures to the Blobstream bridge, or a DAC verifying signatures on a simple L1 contract. In this article I'll focus on alternative blockchains used as the external data availability layer and not on DACs.

One of the most important guarantees than an Optimium needs to inherit from the base layer is the consensus guarantee. Data unavailability of an external network is not attributable internally, so the Optimium must rely on the external network consensus to sign only available blocks. This can be better guaranteed via economic security, since in the external network such faults are attributable and can be appropriately penalized either via double sign slashing or via social consensus. Data unavailability for an Optimium is extremely dangerous since it allows for invalid state roots to be finalized, as challengers wouldn't be able to retrieve the data to prove them wrong.

The alt DA network might also be extremely unstable and reorg very frequently, but as the Optimium inherits the base layer consensus guarantees it wouldn't reorg with it. Something inherits consensus of a layer if it reorgs if and only if such layer reorgs. Instead, if a project reorgs with the DA layer, from the perspective of the other layer it acts like a sidechain.

In practice, the Optimium's derivation rule needs to be implemented in the following way:

1. Read the confirmed data commitments from the base layer (e.g. Ethereum) and use such ordering for deriving the state of the Optimium;
2. Call the DA layer to get the preimages of such commitments to actually compute the state.

{{< figure src="celestia-optimium.png" alt="An optimium using Celestia" position="center" style="border-radius: 8px;" caption="A generic Ethereum Optimium using Celestia for DA. Note that the ordering on Celestia (1-2-3) is different than the one on Ethereum (1-3-2), which is ultimately what gets used." >}}

If the DA layer reorgs, the Optimium doesn't reorg because it follows the base layer ordering, therefore inheriting reorg resistance. The problem arises if the DA bridge accepts a block from the DA layer that is then reorged, since it would then start to follow a non canonical fork. However, even in this case, if the data remains available in the non canonical fork, the Optimium would still function safely, making this the biggest difference in security between an Optimium and a sidechain with a validating bridge (see [this blogpost](https://vitalik.eth.limo/general/2023/10/31/l2types.html) from Vitalik for a deeper explanation, i.e. Validiums vs quasi-Validiums). Despite this, it would be much more fragile since any economic guarantee would be lost and relaying unavailable blocks becomes trivial. Bridges are most often designed to only confirm finalized blocks from the DA layer, i.e. blocks signed by >2/3 of the network stake, but even finalized blocks can be reorged in case of a consensus failure or simply because of social consensus.

{{< figure src="celestia-fork.png" alt="Celestia forking the Optimium off" position="center" style="border-radius: 8px;" caption="An Optimium following a non-canonical DA layer fork." >}}

## The safest Optimium

To summarize, the safety of an Optimium depends on the safety of the DA layer consensus. For this reason, an Optimium should choose the DA layer with the highest economic security and safest consensus. As we all know, Ethereum is the chain that gives out the strongest finality guarantees out there, so why not using that?

An Ethereum Optimium can implement an Ethereum DA bridge by accepting finalized blocks that are signed by >2/3 of the network stake in a smart contract on Ethereum itself. The network can still subject to reorgs of finalized blocks or the majority signing off unavailable blocks, but for Ethereum this is extremely unlikely.

{{< figure src="ethereum-optimium.png" alt="An Ethereum Optimium" position="center" style="border-radius: 8px;" caption="An Ethereum Optimium using Ethereum for DA." >}}

## Conclusion: wait wait wait, isn't this just an Optimistic Rollup?

We often hear than an Optimistic Rollup is a blockchain that inherits consensus from the base layer and publishes DA on the same layer. This little troll exercise was made to test this mental model and to show that it is not enough: an Optimium built this way still relies on Ethereum's consensus majority being honest, which is not the case for an Optimistic Rollup. A Rollup, by definition, operates safely even in the presence of a dishonest majority, in the same way that Ethereum itself does: data unavailability is trivially attributable, so not even a malicious Ethereum majority can fool a Rollup full node, making it much secure than an Optimium.

But if you really want to build an Optimium, just make sure to use the safest DA layer out there 😉 (jk don't do that).
