---
layout: post
title:  "Multi-slot MEV Prevention via FOCIL"
image: "assets/images/4.png"
excerpt: "FOCIL enables multiple proposers to force include transactions into a block. What types of multi-slot MEV does this prevent?"
---

## Description

There is multi-slot MEV when a sophisticated actor can extract more MEV from controlling multiple consecutive slots than the sum of MEV multiple separate actors could extract from separately controlling those same successive slots. FOCIL partially breaks a single actor's control over a slot because it allows multiple proposers to force include transactions into a block. This may prevent specific multi-slot MEV techniques, but it could be costly. What types of multi-slot MEV are prevented by FOCIL, and what are the properties of the prevention in terms of costs, incentive compatibility, and efficiency?

## Resources

- [**Multi Block MEV**](https://collective.flashbots.net/t/multi-block-mev/457/6), Toni Wahrstätter
- [**Multiblock MEV opportunities & protections in dynamic AMMs**](https://arxiv.org/abs/2404.15489), Matthew Willetts, Christian Harrington
- [**Multi-block MEV**](https://arxiv.org/abs/2303.04430), Johannes Rude Jensen, Victor von Wachter, Omri Ross
- [**Inclusion Lists Don’t Fix Multi-Block MEV in APS**](https://x.com/_charlienoyes/status/1806186662327689441), Charlie Noyes
