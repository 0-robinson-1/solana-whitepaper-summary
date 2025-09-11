# Solana White Paper Summary

**Original Title:**
Solana: A new architecture for a high performance blockchain v0.8.14

Author:
Anatoly Yakovenko

**I (Robinson) have studied the original Solana White Paper version 0.8.14 (which was published on GitHub on October 18, 2018 in the solana-labs/whitepaper repository) and have taken notes here to come back to and work on later, feel free to have a read...**

## Table of Contents
- [Abstract](#abstract)
- [1. Introduction](#1-introduction)
- [2. Outline](#2-outline)
- [3. Network Design](#3-network-design)
- [4. Proof of History](#4-proof-of-history)
  - [4.1 Description](#41-description)
  - [4.2 Timestamp for Events](#42-timestamp-for-events)
  - [4.3 Verification](#43-verification)
  - [4.4 Horizontal Scaling](#44-horizontal-scaling)
  - [4.5 Consistency](#45-consistency)
  - [4.6 Overhead](#46-overhead)
  - [4.7 Attacks](#47-attacks)
    - [4.7.1 Reversal](#471-reversal)
    - [4.7.2 Speed](#472-speed)
    - [4.7.3 Long Range Attacks](#473-long-range-attacks)
- [5. Proof of Stake Consensus](#5-proof-of-stake-consensus)
  - [5.1 Description](#51-description)
  - [5.2 Terminology](#52-terminology)

## Abstract

This paper proposes a new blockchain architecture based on Proof of History (PoH) - a prroof for verifying order and passage of time between events. PoH is used to encode trustless passage of time into a ledger - an append only data structure (Append only means new data can only be added to the end of the blockchain, existing data can't be modified, deleted or reordered... This ensures immutability as once semething is added, it stays this way permanent). When used alongside a consensus algorithm such as Proof of Work (PoW) or Proof of Stake (PoS), PoH can reduce messaging overhead in a Byzantine Fault Tolerant replicated state machine, resulting in sub-second finality times.  
This paper also proposes two algorithms that leverage the time keeping properties of the PoH ledger - a PoS algorithm that can recover from partitions of any size an efficient streaming Proof of Replication (PoRep). **The combination of PoRep and PoH provides a defense against forgery of the ledger with respect to time (ordering) and storage.** The protocol is analyzed on a 1 gbps network, and this paper shows that throughput up to 710K transactions per second (TPS) is possible with today's hardware.

## 1. Introduction

Blockchain is an implementation of a fault tolerant replicated state machine (a distributed computing model where multiple identical copies of a system process the same sequence of inputs to maintain consistent states across all instances, while being designed to continue operating correctly even if some replicas fail or behave maliciously).  
Current publicly available blockchains do not rely on time, or make a weak assumption about the participant's abilities to keep time. Each node in the network usually relies on their own local clock without knowledge of any other participants clocks in the network. The lack of a trusted source of time means that when a message timestamp is used to accept or reject a message, there is no guarantee that every other participant in the network will make the exact same choice. The PoH presented here is designed to create a ledger with verifiable passage of time, i.e. duration between events and message ordering. It is anticipated that every node in the network will be able to rely on the recorded passage of time in the ledger without trust.

## 2. Outline

Description about how this paper is organized...

## 3. Network Design

As shown in **Figure 1**, at any given time a system node is designated as Leader to generate a Proof of History sequence, providing the network global read consistency and a verifiable passage of time. The Leader sequences user messages and orders them such that they can be efficiently processed by other nodes in the system, **maximizing throughput**. It executes the transactions on the current state that is stored in RAM and publishes the transactions and a signature of the final state to the replications nodes called **Verifiers**. Verifiers execute the same transactions on their copies of the state, and publish their computed signatures of the state as confirmations. The published confirmations serve as votes for the consensus algorithm.

![Solana Transactions](images/solana-network-design.png)

In a non-partitioned state, at any given time, there is one Leader in the network. Each Verifier node has the same hardware capabilities as a Leader and can be elected as a Leader, this is done through PoS based elections.

## 4. Proof of History

**Proof of History is a sequence of computation that can provide a way to cryptographically verify passage of time between two events**. It uses a cryptographically secure function written so that output cannot be predicted from the input, and must be completely executed to generate the output. The function is run in a sequence on a single core,** its previous output as the current input**, periodically recording the current output, and how many times its been called. The output can then be re-computed and verified by external computers in parallel by checking each sequence segment on a separate core.  
Data can be timestamped into this sequence by appending the data (or a hash of some data) into the state of the function. The recording of the state, index and data as it was appended into the sequences provides a timestamp that can guarantee that the data was created sometime before the next hash was generated in the sequence. This design also supports horizontal scaling as multiple generators can synchronize among each other by mixing their state into each others' sequences.

## 4.1 Description

The system is designed to work as follows. With a cryptographic hash function, whose output cannot be predicted without running the function (e.g. sha256, ripemd,...), run the function from some random starting value and take its output and pass it as the input into the same function again. Record the number of times the function has been called and the output at each call. **The starting random value chosen could be any string, like the headline of the New York times for the day**.  

For example:
#### PoH Sequence
|Index|Operation|Output Hash|
|:---:|:---:|:---:|
|1|sha256("any random starting value")|hash1|
|2|sha256(hash1)|hash2|
|3|sha256(hash2)|hash3|

(Where hashN represents the actual hash output. It is only necessary to publish a subset of the hashes and indices **at an interval**.)

For example:
#### PoH Sequence
|Index|Operation                           |Output Hash|
|:-----:|:-----------------------------------:|:-----------:|
|1|sha256("any random starting value")|hash1|
|200|sha256(hash199)|hash200|
|300|sha256(hash299)|hash300|

As long as the hash function chosen is collision resistant, this set of hashes can only be computed in sequence by a single computer thread. This follows from the fact that there is no way to predict what the hash value at index 300 is going to be without actually running the algorith from the starting value 300 times. It can be inferred from the data structure **that real time has passed between index 0 and index 300**.  

  ## 4.2 Timestamp for Events  

In the example in Figure 2, hash 62f51643c1 was produced on count 510144806912 and hash c43d862d88 was produced on count 510146904064. This proves that we can
trust that real time passed between count **510144806912** and count **510146904064**.  

![Solana Time has passed](images/solana-time-passed.png)

This sequence of hashes can also be used to record that some piece of data was created before a particular hash index was generated. Using a 'combine' function to combine the piece of data with the current hash at the current index. The data can simlpy be a cryptographically unique hash of arbitrary event data.  
The combine function can be a simple append of data, or any operation that is collision resistant. The next generated hash represents a timestamp of the data, because it could have only been generated after that specific piece of data was inserted.

Some external event occurs, like a photograph was taken, or any arbitrary digital data was created:

#### PoH Sequence With Data
| Index | Operation                                  | Output Hash |
|-------|--------------------------------------------|-------------|
| 1     | sha256("any random starting value")        | hash1       |
| 200   | sha256(hash199)                            | hash200     |
| 300   | sha256(hash299)                            | hash300     |
| 336   | sha256(append(hash335, photograph_sha256)) | hash336     |

Hash336 is computed from the appended binary data of hash335 and the sha256 of the photograph. The index and the sha256 of the photograph are recorded as part of the sequence output. So anyone verifying this sequence can then recreate this change to the sequence.

#### PoH Sequence With 2 Events
| Index | Operation                                   | Output Hash |
|-------|---------------------------------------------|-------------|
| 1     | sha256("any random starting value")         | hash1       |
| 200   | sha256(hash199)                             | hash200     |
| 300   | sha256(hash299)                             | hash300     |
| 336   | sha256(append(hash335, photograph1_sha256)) | hash336     |
| 400   | sha256(hash399)                             | hash400     |
| 500   | sha256(hash499)                             | hash500     |
| 600   | sha256(append(hash599, photograph2_sha256)) | hash600     |
| 700   | sha256(hash699)                             | hash700     |

Because the initial process is still sequential, we can then tell that things entered into the sequence must have occurred sometime before the future hashed value was computed. Photograph2 was created before hash600 and photograph1 was created before hash336. Inserting the data into the sequence of hashes results in a change to all subsequent values in the sequence. As long as the hash function used is collision resistant, and the data was appended, it should be computationally impossible to pre-compute any future sequences based on prior knowledge of what data will be integrated into the sequence.  
The data that is mixed into the sequence can be the raw data itself, or just a hash of the data with accompanying metadata. 

![Inserting Data into PoH](/images/solana-inserting-data.png)

In the example in Figure 3, input cfd40df8… was inserted into the Proof of History sequence. The count at which it was inserted is 510145855488 and the state at which it was inserted is 3d039eef3. All the future generated hashes are modified by this change to the sequence, this change is indicated by the color change in the figure. **Every node observing this sequence can determine the order at which all events have been inserted and estimate the real time between the insertions**.

  ## 4.3 Verification

The sequence can be verified correct by a multicore computer in significantly less time than it took to generate it.

For example:
#### Core 1
| Index | Data                                   | Output Hash |
|-------|----------------------------------------|-------------|
| 200   | sha256(hash199)                        | hash200     |
| 300   | sha256(hash299)                        | hash300     |
#### Core 2
| Index | Data                                   | Output Hash |
|-------|----------------------------------------|-------------|
| 300   | sha256(hash299)                        | hash300     |
| 400   | sha256(hash399)                        | hash400     |

![Verification using Multiple Cores](/images/solana-verification-using-multiple-cores.png)

The expected time to verify that the sequence is correct is going to be:  

$$\frac{\text{Total number of hashes}}{\text{Hashes per second per core} \times \text{Number of cores available to verify}}$$  

In the example in Figure 4, each core is able to verify each slice of the sequence in parallel. Since all input strings are recorded into the output, with the counter and state that they are appended to, the verifiers can replicate each slice in parallel. The red colored hashes indicate that the sequence was modified by a data insertion.

  ## 4.4 Horizontal Scaling

It's possible to synchronize multiple Proof of History generators by mixing the sequence state from each generator to each other generator, and thus achieve horizontal scaling of the PoH generator. **This scaling is done without sharding.** The output of both generators is necessarry to reconstruct the full order of events in the system.  

#### PoH Generator A
| Index | Hash   | Data   |
|-------|--------|--------|
| 1     | hash1a |        |
| 2     | hash2a | hash1b |
| 3     | hash3a |        |
| 4     | hash4a |        |

#### PoH Generator B
| Index | Hash   | Data   |
|-------|--------|--------|
| 1     | hash1b |        |
| 2     | hash2b | hash1a |
| 3     | hash3b |        |
| 4     | hash4b |        |

Given generators A and B, A receives a data packet from B (hash1b), which contains the last state from Generator B, and the last state generator B observed from Generator A. The next state hash in Generator A then depends on the state from Generator B, so we can derive that hash1b happened sometime before hash3a. This property can be transitive, so if three generators are synchronized through a single common generator A <-> B <-> C, we can trace the dependency between A and C even though they were not synchronized directly.  
By periodically synchronizing the generators, each generator can then handle a portion of external traffic, thus the overall system can handle a larger amount of events to track at **the cost of true time accuracy** due to network latencies between the generators. A global order can still be achieved by picking some deterministic function to order any events that are within the synchronization window, such as by the value of the hash itself.  
In Figure 5, the two generators **insert each other’s output state and record the operation**. The color change indicates that data from the peer had modified the sequence. The generated hashes that are mixed into each stream are highlighted in bold. The synchronization is transitive. A <-> B <-> C There is a provable order of events between A and C through B. Scaling in this way comes at the cost of availability. 10 × 1 gbps connections with availability of 0.999 would have 0.99910 = 0.99 availability. (Availability is the probability that a system is operational and accessible. Here, each connection has an availability of 0.999 (99.9% uptime, meaning it’s down only 0.1% of the time)).  
--> When you add more connections, the system’s overall availability is the product of the individual availabilities. For 10 connections, each with 0.999 availability, the combined availability is calculated as 0.999^10 ≈ 0.99 (or 99%). This means the system as a whole is slightly less reliable because the more components you add, the higher the chance that at least one fails... Scaling by adding more connections boosts throughput (more data can be processed), but it reduces overall system availability because the failure of any single connection can impact the system. The trade-off is between handling more traffic and maintaining high reliability, while adding more connections increases capacity, it slightly lowers the system’s overall reliability due to the multiplied probabilities of individual connection failures.

  ## 4.5 Consistency

Users are expected to be able to enforce consistency of the generated sequence and make it resistant to attacks by inserting the last observed output of the sequence they consider valid into their input.

![Two Generators Synchronizing](/images/solana-two-generators-synchronizing.png) 

#### PoH Sequence A
| Index | Data   | Output Hash |
|-------|--------|-------------|
| 10    |        |   hash10a   |
| 20    | Event1 |   hash20a   |
| 30    | Event2 |   hash30a   |
| 40    | Event3 |   hash40a   |

#### PoH Hidden Sequence B
| Index | Data   | Output Hash |
|-------|--------|-------------|
| 10    |        |   hash10b   |
| 20    | Event3 |   hash20b   |
| 30    | Event2 |   hash30b   |
| 40    | Event1 |   hash40b   |

A malicious PoH generator could produce a second hidden sequence with the events in reverse order, if it has access to all the events at once, or is able to generate a faster hidden sequence.  
To prevent this attack, each client-generated Event should contain within iself **the latest hash that the client observed from what it considers to be a valid sequence**. So when a client creates the "Event1" data, they should append the last hash they have observed.

#### PoH Sequence A
| Index |                 Data                  | Output Hash |
|-------|---------------------------------------|-------------|
| 10    |                                       |   hash10a   |
| 20    | Event1 = append(event1 data, hash10a) |   hash20a   |
| 30    | Event2 = append(event2 data, hash20a) |   hash30a   |
| 40    | Event3 = append(event3 data, hash30a) |   hash40a   |

When the sequence is published, Event3 would be referencing hash30a, and if it's not in the sequence prior to this Event, the consumers of the sequence know that it's an invalid sequence (Partial Reordering Attack). To prevent a malicious PoH generator from rewriting the client Event hashes, the client can submit a signature of the event data and the last observed hash instead of just the data...

#### PoH Sequence A
| Index |                             Data                               | Output Hash |
|-------|----------------------------------------------------------------|-------------|
| 10    |                                                                |   hash10a   |
| 20    | Event1 = sign(append(event1 data, hash10a),Client Private Key) |   hash20a   |
| 30    | Event2 = sign(append(event2 data, hash20a),Client Private Key) |   hash30a   |
| 40    | Event3 = sign(append(event3 data, hash30a),Client Private Key) |   hash40a   |

Verification of this data requires a signature verification, and a lookup of the hash in the sequence of hashes prior to this one. Verify:  
(Signature, PublicKey, hash30a, event3 data) = Event3  
Verify(Signature, PublicKey, Event3)  
Lookup(hash30a, PoHSequence)  

![Input with a back reference](/images/solana-input-back-reference.png)

In Figure 6, the user-supplied input is dependent on hash **0xdeadbeef…** existing in the generated sequence sometime before it's inserted. The blue top left arrow indicates that the client is referencing a previously produced hash. The client's message is only valid in a sequence that contains the hash **0xdeadbeef…**. The red color in the sequence indicates that the sequence has been modified by the clients data.

  ## 4.6 Overhead

4000 hashes per second would generate an additional 160 kilobytes of data and would require access to a GPU with 4000 cores and roughly 0.25-0.75 milliseconds of time to verify.

  ## 4.7 Attacks
  ## 4.7.1 Reversal

Generating a reverse order would require an attacker to start the malicious sequence after the second event. This delay should allow any non malicious peer to peer nodes to communicate about the original order.

  ## 4.7.2 Speed

Having multiple generators may make deployment more resistant to attacks. One generator could be high bandwidth and receive many events to mix into its sequence, another generator could be high speed low bandwidth that periodically mixes with the high bandwidth generator. --> The high speed sequence would create a secondary sequence of data that an attacker would have to reverse.

  ## 4.7.3 Long Range Attacks

Long range attacks involve acquiring old discarded client Private Keys, and generating a falsified ledger. Proof of History provides some protection against long range attacks. A malicious user that gains access to old private keys would have to recreate a historical record that takes as much time as the original one they are trying to forge. This would require access to a faster processor than the network is currently using, otherwise the attacker would never catch up in history length.  
Additionally, a single source of time allows for construction of a simpler Proof of Replication. Since the network is designed so that all participants in the network will rely on a single historical record of events. **PoRep & PoH together should provide a defense of both space and time against a forged ledger.

  ## 5 Proof of Stake Consensus  
  ## 5.1 Description

This specific instance of Proof of Stake is designed for quick confirmation of the current sequence produced by the PoH generator, for voting and selecting the next PoH generator and for punishing any misbehaving validators. This algorithm depends on messages eventually arriving to all participating nodes within a certain timeout.

  ## 5.2 Terminology  

**bonds** Bonds are equivalent to a capital expense in Proof of Work. A miner buys hardware and electricity and commits it to a single branch in a PoW blockchain. A bond is coin that the validator commits as collateral while they are validating transactions.  
**slashing** The proposed solution to the nothing at stake problem in PoS systems. When a proof of voting for a different branch is published, that branch can destroy the validator's bond. This is an economic incentive designed to discourage validators from confirming multiple branches.  
**super majority**


**WORK IN PROGRESS-STUDYING AS YOU READ :)**
