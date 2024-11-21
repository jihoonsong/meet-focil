---
layout: post
author: soispoke
author_pfp: "assets/profile_pictures/soispoke.jpg"
author_link: "https://ethresear.ch/u/soispoke/summary"
title: "FOCIL, but with transaction hashes"
publish_date: "Oct 31, 2024"
thumbnail: "assets/thumbnails/focil_but_with_transaction_hashes.jpg"
image: "assets/images/focil_but_with_transaction_hashes.png"
excerpt: "In this note, we explore the pros and cons of propagating only transaction hashes instead of full transactions on the consensus layer peer-to-peer network."
---

## Introduction

In the current FOCIL design specified in [EIP-7805](https://ethereum-magicians.org/t/eip-7805-committee-based-fork-choice-enforced-inclusion-lists-focil/21578), IL committee members build and propagate inclusion lists (ILs) filled with full transactions across the consensus layer peer-to-peer (CL P2P) network. ILs can contain up to `8 KB` of transactions. The byte-size limit on ILs and the restriction of IL proposal to a small set of committee members (i.e., `16`) bind the main resource consumed by the propagation of ILs—namely, bandwidth. The advantage of propagating ILs signed by committee members and containing full transactions is that it guarantees the builder (i.e., a proposer doing local block building or an external builder) and attesters will receive the entire transaction information without any further action required from them, thus ensuring reliability. 

However, this approach can be considered wasteful compared to propagating only transaction hashes, considering that an average transaction is about the size of five hashes. In this note, we explore the pros and cons of propagating only transaction hashes instead of full transactions.

## Pros

- **Bandwidth efficiency**: The first, most obvious advantage of propagating ILs with transaction hashes is the significant improvement in bandwidth efficiency. Considering that ILs can contain up to `8 KB` of transactions and an average transaction is about `200 bytes`, each IL can include approximately `40` transactions. However, if we were to propagate only transaction hashes, which are about `32 bytes` each, the same IL could include around `256` hashes.
- **Mempool Flooding:** If IL committee members send ILs containing full transactions, a malicious proposer could crowd the ILs by sending a single calldata-heavy transaction (e.g., `8 KB`) that pays very high priority fees into the mempool. IL committee members would most likely include this transaction in their ILs if they use priority fee ordering to build them, crowding out other valid transactions pending in the mempool. If the proposer was the one flooding the mempool, it could even invalidate its `8 KB` transaction by first emptying the balance of the Externally Owned Account (EOA) that sent it, causing the large transaction to fail due to insufficient funds. There are ways to mitigate such attacks by relying on modified inclusion rules (see more details [here](https://hackmd.io/@fradamt/focil-flooding)), but it involves being opinionated about what local rules we want proposers to use instead of letting them chose based on their preferences.
    
    On the other hand, if ILs only contain transaction hashes, there is no way a transaction can consume more than its `32 bytes` hash. By specifying that ILs are allocated a fixed number of slots (e.g., `256`) for transactions that consume a minimum of `9000 gas` (i.e., the cost of a balance-editing `CALL`), an attacker would need to send `256` transactions to successfully flood the mempool. This approach makes such an attack prohibitively costly. To address cases where the proposer can invalidate their own transactions, we can implement a rule that *ILs should include no more than one transaction per EOA (as mentioned in [@fradamt’s note](https://hackmd.io/@fradamt/focil-flooding))*. This means the proposer would have to include a corresponding number of transactions in the payload to stuff IL slots, and pay the cost of executing these invalidating transactions. 
    
- **Forward blob compatibility:** Given the current state of things, using transaction hashes would allow us to support blob transactions immediately. We could treat blob transactions as normal transactions since there is a single mempool, and nodes have to download entire blobs anyway. However, it is worth noting that adopting [PeerDAS](https://ethresear.ch/t/peerdas-a-simpler-das-approach-using-battle-tested-p2p-components/16541) and moving towards a sharded mempool would require some design changes. Since nodes wouldn't download full blob transactions and would only collect samples, we would need to provide them with an availability signal to ensure they can confidently include blob transactions in their ILs without having full knowledge of whether the blobs are available or not.

## Cons

The main drawback of using transaction hashes lies in the fact that there is no way to ensure that ILs broadcast on the CL P2P network can be enforced. Since the ILs don't contain full transactions, nodes would have to check whether a transaction hash corresponds to an available transaction by querying the execution layer. Let’s dive a bit deeper to see whether this issue can be circumvented:

- **Good case**: In the good, normal-case scenario, all we need to do is allow a bit more time for the builder to send a request to the EL and determine whether the transaction hashes from the ILs correspond to available transactions. This is assuming IL transactions are available in most local mempools (i.e., the builder’s and attesters’ local mempools).
- **Bad case**: In the bad-case scenario, a transaction hash included in an IL does not correspond to a full transaction available to the builder. However, it's important to note that a transaction absent from the builder's local mempool is most likely also missing from the majority of the attesters' local mempools. In this case, the inclusion of that particular transaction is not enforced, and the builder's block will be considered to have satisfied the IL conditions.
    
    In a nutshell, if we assume no adversarial behavior, the full transaction is either widely available—allowing the builder to easily find it in their local mempool or retrieve it by asking a few peers (e.g., using something like [`GetPooledTransactions`](https://github.com/ethereum/devp2p/blob/master/caps/eth.md#getpooledtransactions-0x09) requests)—or it's not widely available, meaning that most attesters (i.e., more than half) won't be able to retrieve it either. In that case, the transaction will not be enforced but IL conditions will be satisfied.
    
- **Adversarial case**: In an adversarial scenario, an attacker could attempt to trick the builder into being unable to retrieve the corresponding full transaction while more than half of the attesters can. This situation, in practice, shouldn't occur if the proposer is given extra time (e.g., an additional `3 seconds`) compared to the attesters to retrieve the transaction. However, the attack would have more chances to succeed if the transaction was replaced or evicted from the mempool right after the attesters’ view-freeze deadline had passed.
    
    To prevent this, we would need the EL to **(1)** Prioritize requesting IL transactions available in the mempool and **(2)** "lock" transactions pending in the mempool once they have received transaction hashes from ILs broadcast over the CL P2P network. This locking mechanism would ensure that these transactions can't be replaced or evicted until **`t=0s`** of **`slot n+1`**, or until the proposer has proposed its block to the rest of the network.
    

## Proposed mechanism

In this section, we propose a candidate design for FOCIL using transaction hashes and outline how these changes affect roles and participants. 

### **IL Committee Members**

The role of IL committee members remains largely the same. The main difference is that they now include up to `256` transaction hashes in their ILs instead of full transactions before broadcasting them over the CL P2P network.

### **Validators**

- Validators receive ILs with transaction hashes from the P2P network. They store all new ILs that pass the CL P2P validation rules, and any evidence of IL equivocation by committee members (i.e., if multiple ILs are received from the same committee member) until the view freeze deadline at **`t=8s`**. Note that the view freeze deadline is now `1s` earlier than in EIP-7805, so we allow more time for the builder to retrieve full transactions from the EL and include the available ones in its payload.
    
    The CL P2P validation rules remain the same in FOCIL with full transactions, except for the size of the IL, that is now defined by the number of slots (i.e., `256`) instead of its size in `KB`. 
    
- After the view freeze deadline at**`t=8s`**during **`slot N`**, validators stop storing new ILs, but continue forwarding ILs to peers and record evidence of equivocation until the attestation deadline at **`t=4s`** during **`slot N+1`**.
- After the attestation deadline of **`Slot N+1`, `t=4s`**, validators ignore any new ILs related to the previous slot's IL committee, and stop recording equivocation evidence for the previous slot's ILs.

### **Builder**

- **`Slot N`, `t=0s to 11s`**: The builder (i.e., a proposer doing local block building or an external builder) receives ILs from the P2P network, forwarding and storing those that pass the CL P2P validation rules.
- **`Slot N`, `t=11s`**:
    - The builder freezes its view of ILs.
    - The builder sends an EngineAPI request to the EL, to verify that transaction hashes in its stored ILs correspond to available transactions pending in the mempool.
    - If any transactions can’t be retrieved from the EL, the builder sends [`GetPooledTransactions`](https://github.com/ethereum/devp2p/blob/master/caps/eth.md#getpooledtransactions-0x09) requests to a few peers in an attempt to recover the missing transactions.
    - The builder then asks the EL to update its execution payload by adding all retrieved transactions.

### Proposer

The proposer, just like in EIP-7805, broadcasts its block with the up-to-date execution payload satisfying IL transactions over the P2P network at **`t=0s`** for **`slot N+1`.**

### Attesters

- **`Slot N+1`, `t=4s`**: Attesters monitor the P2P network for the proposer’s block. Upon detecting it, they:
    - Verify whether all transaction hashes retrieved from their stored ILs are included in the proposer's execution payload, except for ILs whose sender has equivocated. Based on their frozen view of the ILs from **`t = 8s`** in the previous slot, attesters check if the execution payload satisfies IL conditions. This is done either by confirming that all transaction hashes are present or by determining if any missing hashes:
        - Do not correspond to ***available*** transactions pending in the mempool. Attesters must send an `EngineAPI` request to the EL to check whether they are able to retrieve transactions corresponding to the missing hashes. If they can’t retrieve these transactions, they do not enforce them.
        - Do not correspond to ***valid*** transactions when appended to the end of the payload. In such cases, attesters use the EL to perform nonce and balance checks to validate the missing transactions and check whether there is enough space in the block to include the transaction(s).

## EL changes

As mentioned earlier, for the mechanism to be robust the EL needs to treat mempool transactions corresponding to observed IL hashes differently than others by:

- Ensuring these transactions can be easily and promptly retrieved when requested from nodes to perform availability or validity checks
- Lock these transactions so they can’t be replaced or evicted from the mempool until **`t=0s`** of **`slot n+1`**, or until the proposer has broadcast its block with the updated payload over to the network.
