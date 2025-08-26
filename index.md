# Solana White Paper Summary

**Original Title:**
Solana: A new architecture for a high performance blockchain v0.8.14

Author:
Anatoly Yakovenko

**I (Robinson) have studied the original Solana White Paper version 0.8.14 (which was published on GitHub on October 18, 2018 in the solana-labs/whitepaper repository) and have taken notes here to come back to and work on later, feel free to have a read...**

## Table of Contents
[Abstract](#abstract)
1. [Introduction](#introduction)
2. [Outline](#outline)

## Abstract

This paper proposes a new blockchain architecture based on Proof of History (PoH) - a prroof for verifying order and passage of time between events. PoH is used to encode trustless passage of time into a ledger - an append only data structure (Append only means new data can only be added to the end of the blockchain, existing data can't be modified, deleted or reordered... This ensures immutability as once semething is added, it stays this way permanent). When used alongside a consensus algorithm such as Proof of Work (PoW) or Proof of Stake (PoS), PoH can reduce messaging overhead in a Byzantine Fault Tolerant replicated state machine, resulting in sub-second finality times.  
This paper also proposes two algorithms that leverage the time keeping properties of the PoH ledger - a PoS algorithm that can recover from partitions of any size an efficient streaming Proof of Replication (PoRep). **The combination of PoRep and PoH provides a defense against forgery of the ledger with respect to time (ordering) and storage.** The protocol is analyzed on a 1 gbps network, and this paper shows that throughput up to 710K transactions per second (TPS) is possible with today's hardware.

## 1. Introduction
