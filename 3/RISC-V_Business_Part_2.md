---
layout: default
title: RISC-V Business Part 2
author: Ramshreyas Rao
---
# RISC-V Business part 2: The Road to Real-time Proving

!["The Road"](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/00006_4272034470_0000_by_somesh7322_dgqlxcb-pre.jpg)

[Image by Somesh](https://www.deviantart.com/somesh7322/art/00009-969351387-0000-1012953609)

In [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg) of this series, we explored at a high level how the adoption of RISC-V as an open standard is likely to be the foundation on which the [programmable cryptography](https://0xparc.org/blog/programmable-cryptography-1) future of Ethereum will be built, using [Glue and Coprocessor architectures](https://vitalik.eth.limo/images/gluecp/BJdoaxqo0.png). We then used Risc0's VM to 'host' a simple piece of external code, executed it, and then produced a proof that it was executed correctly, to get a concrete, albeit simple caricature of this paradigm.

In Part 2, we will dive deeper first with an overview of [the recently launched and amazing ETHProofs.org](https://ethproofs.org), followed by a hands-on demonstration of proving an Ethereum block using Zeth, Risc0's zkEVM implementation. This should give us a sense of all the bits and pieces that go into creating an Ethereum proof, and also how incredibly long it takes to produce it! 

Once we have the lay of the land, we can proceed to lay out the road to real time proving and discuss how RISC-V and especially its extentions can get us there.

---

### The ZK Schelling point with Ethproofs.org

> Ethproofs is a block proof explorer for Ethereum. It aggregates data from various zkVM teams to provide a comprehensive overview of proven blocks, including key metrics such as cost, latency, and proving time.
> 
> Users can compare proofs by block, download them, and explore various proof metadata(size, clock cycle, type) to better understand the individual zkVMs and its proof generation process.

Ethproofs aggregates the proving time for multiple proof providers (currently [Snarkify](https://ethproofs.org/prover/1a13f366-1a1a-455e-8d82-9516b32dda75), [Succinct](https://ethproofs.org/prover/789bd1d9-158a-4dc7-bb9e-e0f12efda0d1) and [ZKM](https://ethproofs.org/prover/00a4d8ed-9272-42f8-a716-af1eead50de9)) and reports on all their performance parameters, highlighting the best one. The proofs for each can be downloaded and verified - which we will indeed do a little further down in the article. These provers are being run as single instances in comparable and non-optimized deployments for all providers purely for the purposes of benchmarking and comparison - so only one out of every hundred blocks is being tested.

![EthProofs](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/ethproofs.png)

In the image above from the landing page of the website, for each block that is proven, we can see multiple metrics:

1. **Gas used**: This is the total gas that was executed in that particular block, and is a proxy for the computational effort required to execute said block, and therefore the effort required to prove it. The bar gives an indication of its percentage of the gas limit per block.
2. **Cost per proof**: This is an AWS-equivalent, hourly US dollar price estimate for the cost of producing the proof using hardware most similar to that being used by competing provers. The numbers for the best are shown with the average displayed below.
3. **Cost per Mgas**: This is a US dollar estimate of the cost of proving 1 Mgas worth of compute, using the two estimate above.
4. **Proving time**: The time taken to produce the proof of the given block
5. **Proof status**: How many of the provers successfully completed the proof, are in progress, or are in queue to begin proving

To understand this better, let us identify a block with almost complete gas limit utilization - [21728900](https://etherscan.io/block/21728900) (which will be buried in the history when you read this) and dig deeper on [Ethproofs.org](https://ethproofs.org/block/21728900) to get a sense of what the current state of the art (at the time of writing) is for block proving:

![A full block](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/fullblock.png)