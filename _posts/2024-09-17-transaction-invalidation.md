---
layout: post
author: Julian Ma
author_pfp: "assets/profile_pictures/julian_ma.jpg"
author_link: "https://ethresear.ch/u/julian/summary"
title: "Transaction Invalidation"
publish_date: "Sep 17, 2024"
thumbnail: "assets/thumbnails/transaction_invalidation.jpg"
image: "assets/images/transaction_invalidation.png"
excerpt: "Other transactions in the execution payload could invalidate transactions in inclusion lists. Invalid transactions should not be written on-chain because it would expose free data availability (DA). How is free DA prevented in FOCIL? How does FOCIL prevent an adversary stuffing the IL with invalid transactions?"
---

Inclusion list committee members select transactions based on their view of the mempool and previous blocks. They try to choose only transactions that will be valid in the blocks to which the inclusion lists apply. Committee members must commit to their inclusion lists before the block is published. They cannot be sure whether a transaction is valid because of the following two reasons:

1. The block producer may order transactions from the inclusion list anywhere in the block, so a transaction from the inclusion list may be invalidated by a transaction included by the block producer before this transaction.
2. The block the inclusion list applies to may choose a different parent block than the committee member had expected, or the committee member did not have enough time to execute the previous block. Therefore, the transaction a committee member thought was valid could have been invalidated by an earlier block.

Invalid transactions cause problems if they land on the chain because they get free data availability. The transaction does not pay the base fee, but it is now provable that the data committed to in the invalid transaction is available on Ethereum. FOCIL prevents free data availability by not writing the inclusion list on-chain. 

Attesters enforce the inclusion list based on their view of broadcasted inclusion lists. If the block does not satisfy the attesters’ view of the inclusion list, the block has no fork-choice viability in the current slot. In principle, all transactions in broadcasted inclusion lists must be included in the block. However, if a transaction cannot be validly appended to the end of the provided execution payload, it does not have to be included. Attesters check whether a transaction can be validly appended by running the `Valid` function, which at this point is just the usual transaction validity check on the sender’s nonce and balance, but which could be arbitrary (but cheap to execute) EVM code in a full AA world. Because attesters check whether the inclusion list has been satisfied based on their local view and not as part of the state transition function, there is no need for on-chain information about the inclusion list, and in particular, no need for full transactions to go on-chain, thus preventing free DA.

Another way invalid transactions can cause problems is if many transactions in the inclusion list are invalid. This means the inclusion list does not do a good job of surfacing those transactions that would benefit from being included. FOCIL combats this by the inclusion rule committee members run to construct their inclusion lists.

First, committee members try to execute the previous execution payload before constructing their inclusion list. This helps them see which transactions are invalid based on the poststate of the last block. Secondly, committee members are free to choose any inclusion rule that they believe best selects transactions that benefit from being in the inclusion list. For example, committee members could choose transactions based on how long they have been in the mempool, their priority fee, a combination of the two, or any other [heuristic](https://x.com/arindamsingh03/status/1838541552387264673?s=61) they believe is valuable. 

FOCIL should be robust to an adversary [stuffing](https://ethresear.ch/t/fun-and-games-with-inclusion-lists/16557) the inclusion list with invalid transactions. For example, if all inclusion list committee members build their inclusion lists by selecting those transactions that pay the highest priority fee, then an adversary could send high-paying valid transactions in the mempool and invalidate them in the execution payload the inclusion list constrains. This attack would allow an adversary to stuff the inclusion lists for free.

Using a diversity of inclusion rules would prevent this attack. If some committee members select transactions based on priority fees, whereas others choose transactions based on time in the mempool, stuffing the inclusion list becomes very difficult. To stuff a inclusion list of a committee member that selects transactions based on time in the mempool, an adversary needs to control multiple slots in a row since it must ensure that transactions are valid in the mempool but would be invalidated in a block to prevent costs. Moreover, these inclusion rules would be local rules that a committee member or client could easily change.

## Description

The aggregate inclusion list does not constrain the ordering of the execution payload, which could mean that a transaction in the execution payload invalidates a transaction in the aggregate inclusion list. FOCIL prevents free DA by never posting complete transactions on-chain, similar to the [no free lunch design](https://ethresear.ch/t/no-free-lunch-a-new-inclusion-list-design/16389). However, an adversary could stuff the inclusion list with transactions invalidated in the execution payload. This attack would be cheaper than [regular block-stuffing techniques](https://ethresear.ch/t/fun-and-games-with-inclusion-lists/16557) since invalid transactions do not go on-chain and do not pay base fees. How can the effectiveness of FOCIL be maximized in the presence of such an adversary?

## Possible solutions

- Inclusion list committee members could employ inclusion rules that prevent invalid transaction stuffing attacks.
- FOCIL could enforce that other transactions in the execution payload may not invalidate transactions from the aggregate inclusion list.
- Decreasing the time window in which it is certain who the next execution proposer is to such an extent that inclusion list committee members create ILs before the next execution proposer is known.

## Resources

- [**Vryx: Fortifying Decoupled State Machine Replication**](https://hackmd.io/@patrickogrady/rys8mdl5p#Making-the-Case-for-Decoupled-State-Machine-Replication-DSMR), Patrick O’Grady
- [**No free lunch - a new inclusion list design**](https://ethresear.ch/t/no-free-lunch-a-new-inclusion-list-design/16389), Mike Neuder
- [**Fun and games with inclusion lists**](https://ethresear.ch/t/fun-and-games-with-inclusion-lists/16557), Barnabé Monnot
