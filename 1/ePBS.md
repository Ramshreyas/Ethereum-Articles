---
layout: default
title: ePBS
---

# Enshrined Proposer-Builder Separation

*October 7, 2024*

## Table of Contents

1. [What is ePBS?](#what-is-epbs)
2. [Latest Updates and Developments](#latest-updates-and-developments)
    - [Recent Articles and Publications](#recent-articles-and-publications)
    - [Community Discussions](#community-discussions)
    - [Technical Proposals and Repositories](#technical-proposals-and-repositories)
3. [Motivations Behind Current Choices](#motivations-behind-current-choices)
    - [Security Enhancements](#security-enhancements)
    - [MEV Mitigation Strategies](#mev-mitigation-strategies)
    - [Decentralization Goals](#decentralization-goals)
4. [Future Outlook and Next Steps](#future-outlook-and-next-steps)
    - [Upcoming Proposals](#upcoming-proposals)
    - [Potential Challenges](#potential-challenges)
    - [Opportunities for Contribution](#opportunities-for-contribution)
5. [Conclusion](#conclusion)
6. [References](#references)

---

## What is ePBS? 

**[skip](#latest-updates-and-developments)**

For every slot in Ethereum, a previously psuedo-randomly chosen validator has [a monopoly on proposing the next block](https://www.blocknative.com/blog/anatomy-of-a-slot). This validator bundles the transactions into a block, runs the transactions to ensure validity, and then proposes the block to a (psuedo-randomly selected) committee of attestors who validate the block for inclusion. This monopoly on block-building opens up the opportunity to order, include, exclude and indeed introduce transactions so as to make a profit for the validator (and builders who do this for them) over and above fees and issuance. This is the source of 'MEV', and this context is well understood. 

Since this is a highly sophisticated activity, a complex [MEV](https://www.paradigm.xyz/2021/02/mev-and-me) supply-chain dominated by a few off-chain actors has emerged, where [builders](https://www.paradigm.xyz/2023/04/mev-boost-ethereum-consensus) bundle and 'sell' high-value blocks to block [proposers](https://www.paradigm.xyz/2023/04/mev-boost-ethereum-consensus) (validators) via [relays](https://www.paradigm.xyz/2023/04/mev-boost-ethereum-consensus) for a share of the profits - [and is in fact responsible for the majority of transactions included in Ethereum today](https://www.theblock.co/data/on-chain-metrics/ethereum/percentage-of-blocks-proposed-by-mev-boost-relay). The interactions between these parties are currently out-of-protocol, and rely on multiple trust assumptions.

Builders submit hash tree roots (and a bid to be chosen) to relays for an execution payload that they promise to deliver upon being selected by a proposer. The proposer polls the relay, selects the best bid [as it gets close to the end of the block](https://www.blocknative.com/blog/anatomy-of-a-slot#7) and submits a [SignedBlindedBeaconBlock](https://github.com/ethereum/builder-specs/blob/main/apis/builder/blinded_blocks.yaml#L19) to the relay, trusting that the HTR will be replaced with a full execution payload from the builder in time to broadcast the complete block for inclusion.

This system is problematic because it requires the proposer to trust the relay to:
- Relay the blinded block to the builder.
- Ensure the builder provides a valid payload matching the HTR.
- Not manipulate the block or steal MEV.
- Handle payments correctly.

The builder is trusting that the relay:
- Transmits their bid accurately
- Enforces the payment from the validator
- Will not unbundle the block and manipulate its contents

There are also some inefficiencies related to propagating the blocks and the risk of reorgs which will be discussed below.

The primary proposal on the Ethereum Roadmap to mitigate this is [(enshrined) Proposer Builder Separation - EIP 7732](https://eips.ethereum.org/EIPS/eip-7732). In this paradigm, the currently 'informal', offchain 'builder' role is formally recognized within the protocol and the interaction between the proposer and the builder is disintermediated (no relay) and specified fully. 

(Click the arrows to see the flow. Select the current form of PBS or proposed ePBS flow from the legend to compare)

<iframe src="https://ramshreyas.github.io/Possible-Futures/viewer.html?data=graphs/ePBS.json" width="100%" height="1000px" style="border:none;"></iframe>

ePBS:
- **Builder stakes**: Builders have to stake on the Beacon chain to participate
- **Builder Prepares Bid**: Builders assemble execution payloads (ExecutionPayloadEnvelope) and prepare SignedExecutionPayloadHeader objects, containing bid values and commitments to their payloads.
- **Bid Submission**: Builders submit their bids directly to proposers or broadcast them on a P2P network. This direct interaction reduces reliance on relays.
- **Proposer Selects Bid**: Proposers collect bids and choose the highest one based on value and other factors.
- **Proposer Includes Header in Beacon Block**: The proposer incorporates the chosen builder's SignedExecutionPayloadHeader into their beacon block. This acts as an on-chain commitment to include that specific payload.
- **Block Proposal and Payment**: The proposer broadcasts the beacon block, initiating the payment process. The builder's staked ETH is automatically reduced by the bid amount on the beacon chain, ensuring payment to the proposer.
- **Builder Reveals Payload**: Upon seeing their header included in a valid beacon block, the builder reveals their full ExecutionPayloadEnvelope, containing the transactions and data for the block.
Payload Validation: Validators independently validate the received payload against the committed header in the beacon block. If it's valid and timely, as attested by the PTC, it's likely included in the canonical chain.
- **Payload Timeliness Committee (PTC)**: A dedicated committee (PTC) attests to the timely presence of the builder's payload, providing an additional layer of security and helping to prevent withholding or manipulation by builders or proposers.


## Latest Updates and Developments

### Recent Articles and Publications

Summarize the most recent articles, blog posts, and research papers on ePBS. Include direct links to these sources for reference.

- **Article Title 1**  
  *Publication Name*  
  [Read more](https://example.com/article1)

- **Research Paper 1**  
  *Authors*  
  [Download PDF](https://example.com/paper1)

### Community Discussions

Highlight key discussions from platforms like Discord, Ethereum forums, and other relevant communities. Provide links to specific threads or channels where ePBS is being actively discussed.

- **Discord Channel:** Overview of ongoing conversations and key takeaways.  
  [Join the Discussion](https://discord.com/invite/example)

- **Forum Thread:** Summary of debates and consensus points.  
  [View Thread](https://forum.ethereum.org/t/epbs-discussion/12345)

### Technical Proposals and Repositories

List the latest technical proposals, GitHub repositories, and code contributions related to ePBS.

- **EIP-XXXX: Encrypted PBS Specification**  
  [EIP Details](https://eips.ethereum.org/EIPS/eip-xxxx)

- **GitHub Repository:** Active development and codebase for ePBS implementation.  
  [View Repository](https://github.com/ethereum/epbs)

## Motivations Behind Current Choices

### Security Enhancements

Explain how the current ePBS implementations aim to bolster Ethereum's security. Discuss specific vulnerabilities addressed and the mechanisms introduced to mitigate them.

- **Encrypted Communication:** Benefits and security implications.
- **Verification Protocols:** How proposers verify block validity without accessing transaction data.

### MEV Mitigation Strategies

Detail the strategies employed by ePBS to reduce **Maximal Extractable Value (MEV)** exploitation. Explain why these strategies are crucial for maintaining network integrity.

- **Transaction Privacy:** Preventing proposers from reordering or censoring transactions.
- **Fair Block Construction:** Ensuring builders compete fairly without undue influence.

### Decentralization Goals

Discuss how ePBS contributes to Ethereum's decentralization objectives. Highlight measures taken to prevent centralization of power among builders or proposers.

- **Role Separation:** Benefits of distinct roles for proposers and builders.
- **Competition and Fairness:** Encouraging a diverse set of builders to participate.

## Future Outlook and Next Steps

### Upcoming Proposals

Outline the forthcoming proposals and enhancements planned for ePBS. Mention any EIPs in the pipeline or scheduled updates.

- **EIP-YYYY:** Description and expected impact.  
  [EIP Details](https://eips.ethereum.org/EIPS/eip-yyyy)

### Potential Challenges

Identify and analyze the challenges that ePBS might face in the near future. This could include technical hurdles, adoption barriers, or unforeseen security issues.

- **Scalability Concerns:** Handling increased network load.
- **Implementation Complexity:** Integrating ePBS with existing Ethereum infrastructure.

### Opportunities for Contribution

Encourage core developers to engage with ePBS development. Highlight areas where contributions are needed, such as code development, security audits, or community education.

- **Open Issues on GitHub:** [View Issues](https://github.com/ethereum/epbs/issues)
- **Community Working Groups:** Participate in focused discussions and task forces.

## Conclusion

Provide a succinct summary of ePBS's current state, its importance for Ethereum's future, and the collective efforts required to advance its implementation. Reiterate the key points discussed in the article.

## References

List all the sources referenced in the article for further reading and verification.

- [ePBS Whitepaper](https://example.com/epbs-whitepaper)
- [PBS Overview](https://ethereum.org/en/developers/docs/consensus-mechanisms/pbs/)
- [Discord Channel](https://discord.com/invite/example)
- [GitHub Repository](https://github.com/ethereum/epbs)

---
