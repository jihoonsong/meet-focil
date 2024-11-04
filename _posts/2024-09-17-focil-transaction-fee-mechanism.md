---
layout: post
title:  "FOCIL Transaction Fee Mechanism"
image: "assets/images/2.png"
excerpt: "What is the best way to charge fees and distribute these fees for inclusion in the inclusion list? How should “best” be defined?"
---

## Description

In Ethereum, users pay fees to get their transactions on-chain. The amount of fees they pay and how these fees are distributed are determined by a transaction fee mechanism (TFM). Ethereum currently uses the TFM proposed in [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559), which is [characterized by](https://timroughgarden.org/papers/eip1559.pdf) improved user experience, incentive compatibility for myopic block producers, and resistance to out-of-band agreements between users and block producers. FOCIL provides a new way for transactions to go on-chain. Can FOCIL be enhanced with a TFM? If so, what are the explicit goals the TFM should optimize for?

## Questions

- Should there be a separate fee for inclusion in the inclusion list?
- What should be the goals of a FOCIL TFM design?
- Can these goals be reached simultaneously?
- How do the properties of inclusion lists, like conditional or [unconditional](https://ethresear.ch/t/unconditional-inclusion-lists/18500), affect the TFM?
- Would FOCIL benefit from a [conditional tip](https://arxiv.org/abs/2301.13321)?

## Resources

- [**Transaction Fee Mechanism Design for the Ethereum Blockchain:
An Economic Analysis of EIP-1559**](https://timroughgarden.org/papers/eip1559.pdf), Tim Roughgarden
- [**Censorship Resistance in On-Chain Auctions**](https://arxiv.org/abs/2301.13321), Elijah Fox, Mallesh Pai, Max Resnick
    