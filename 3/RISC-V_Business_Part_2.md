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

### The ZK Schelling point - ETHProofs.org

> Ethproofs is a block proof explorer for Ethereum. It aggregates data from various zkVM teams to provide a comprehensive overview of proven blocks, including key metrics such as cost, latency, and proving time.
> 
> Users can compare proofs by block, download them, and explore various proof metadata(size, clock cycle, type) to better understand the individual zkVMs and its proof generation process.

