---
layout: post
title:  "Local IL Size"
image: "assets/images/3.png"
excerpt: "Local inclusion lists must likely be bounded in size to prevent excessive bandwidth usage. How do we determine the optimal size of the inclusion lists?"
---

## Description

Local inclusion lists can likely not be unbounded since they can include invalid transactions and must be propagated and processed by various actors in Ethereum. Unbounded local inclusion lists would, therefore, open a DoS vector. The main resource that local inclusion lists consume is bandwidth. How should the size of a local inclusion list be defined, and what is the optimal size for a local inclusion list?

## Questions

- How should the size of a local inclusion list be determined?
    - This is important because the units of gas a transaction will use are unknown at the time it is included in a local inclusion list. Should the maximum gas limit of a transaction be used?
- What is the optimal size of a local inclusion list?
- How does this depend on the various properties of inclusion lists?

## Resources

- https://hackmd.io/@ttsao/focil-resource-considerations
- https://ethresear.ch/t/aucil-an-auction-based-inclusion-list-design-for-enhanced-censorship-resistance-on-ethereum/20422