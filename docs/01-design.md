# Design

## Introduction

The ssb-clock feature aims to address the issues of feed forks and incomplete
causality graphs in Secure Scuttlebutt (SSB). It does so by introducing a new
data structure, the Bloom clock [@abs-1905-13064][@Kshemkalyani2020TheBC][@Misra2020TheBC], which allows agents to 
express the causality
between messages in potentially in different feeds.
Additionally, the ssb-clock feature provides protocols for efficiently sharing
status and synchronizing feeds between connected peers, resolving forks in a
feed, and establishing a consensus on elapsed time. These protocols aim to
improve the security, reliability, and scalability of SSB by providing a way for
agents to more accurately and efficiently determine the order of messages in a
feed and the state of the system. The ssb-clock feature also aims among
other to improve efficiency and the security of the gossip protocol of SSB by
optimizing the exchange of information between connected peers and
obfuscating the information exchanged.



## Protocols for messages causality

The ssb-clock feature introduces a new data structure, the Bloom clock, to
express the causality between messages within a feed in addition of the
"previous" hash field of the SSB proctol or in different feeds in response to a
given message. The Bloom clock is a counting bloom filter data structure that
allows an agent to update its knowledge of an entity and the version of that
entity in
its database based on update messages received from other agents.

The Bloom clock has two parameters, N and k, which determine its size and the
number of indices produced by the Random Oracle Function (ROF), respectively.
These parameters can be application-dependent and have optimal values that
depend on the application and the chosen trade-off between the size of the
clock, the computational effort required to process it, and the probability of
false responses to queries.

The base rules for updating the Bloom clock are as follows:

The initial Bloom clock of any identifiable SSB entity is all 0. The default
values of N and k and how they can be modified in some cases are specified in
the section on default parameters.

* When an agent produces an update message (UM) on some identifiable entity
  (IE) with a current version BC(n), it produces a BC(n+1) by incrementing the k
  counters at the indices produced by the ROF(UM). The UM is added to the
  agent's feed along with BC(n+1) and the reference to the identifiable entity
  IE.
* When an agent receives an update message UM with the Bloom clock BC(x) on
  some identifiable entity IE with a current version BC(n), it updates its
  knowledge about the IE and updates the version of the IE in its database to
  BC(n+1) = [max(BC(n)[i], BC(x)[i]) for i in 0..(N-1)].

The Bloom clock is a useful tool for expressing the causality between events, or
messages, in a distributed system. It consists of a bit array, or vector, that
is modified by the addition of elements, or hashes, representing events. When
two events, A and B, are compared, if all elements of the Bloom clock of event
A (BC(A)) are pairwise smaller or equal to all elements of the Bloom clock of
event B (BC(B)), then it can be inferred that event A occurred before event B.

Additionally, the "distance" between BC(A) and BC(B) can be used to estimate the
probability of a false positive comparison. The larger the distance between the
two Bloom clocks, the higher the probability of a false positive.

One interesting property of the Bloom clock is that it is independent of the
order of branches when merged. This means that the resulting Bloom clock will be
the same regardless of the order in which the branches were merged.

Another property of the Bloom clock is that it supports efficient removal of
events. By keeping track of the hashes that were added to the Bloom clock, it is
possible to efficiently remove events by removing their corresponding hashes.

Finally, the Bloom clock can be used to estimate the number of events that
occurred between two events in a chain. Given the Bloom clocks at each event, it
is possible to estimate the number of events that occurred between them.

The Bloom clock is encoded in a compact format to allow for efficient storage
and transmission. The encoding is designed to support different block sizes and
formats to allow for flexibility in the trade-off between size and
computational effort.

Base functions for comparing, updating, and serializing Bloom clocks are
provided to facilitate the implementation of the ssb-clock feature. These
functions are described in more detail in the following sections.

## Generic protocol for synchronizing Abstract Entities in SSB

In the context of the ssb-clock feature, an Entity is a data structure that
represents the state of a system at a given point in time. This state is the
result of a chain of Events, where each Event represents a modification to the
Entity and contains a reference to the previous Event in the chain.

An Event is a SSB message that is produced by an agent and added to its feed. It
contains a reference to the previous Event in the chain related to the Entity.
Using the Bloom Clock, an agent associates each Event with a Bloom Clock Stamp,
which represents the version of the Entity after the Event is applied.

The Bloom Clock Stamp is a data structure that allows agents to express the
causality between events and determine probabilistically the order in which they
occurred. In parallel, the back references of an Event allow agents to determine
the order of events deterministically (or with such high probability that it is
considered deterministic).

The Bloom Clock Stamp is not written in the SSB message, but is computed from
the Bloom Clock Stamp of the previous Events in the chain based on a Random
Oracle Function (ROF) that is application dependent.

In the context of the ssb-clock feature, a fork in the chain of Events can be
resolved by applying a Join Strategy, which is a set of rules or algorithms that
determine how the effects of the branches should be merged. The Join Strategy is
application-specific and may involve manual or automatic techniques. An agent
that is aware of a fork MUST apply a Join Strategy defined for the specific
protocol if it wants to produce a new Event about that Entity. This new Event is
called a Join event. Specifically, a Join event in SSB is a message that
contains a reference to the tip Events of all the branches of the fork it is
aware of, and may also contain additional modifications to the Entity. The
associated Bloom Clock Stamp represents the version of the Entity after the Join
Strategy has been applied and any optional modifications have been applied.

The following abstract protocol describes how agents can use the Bloom Clock
Stamp to sync their mutual knowledge of an Entity. The protocol is based on the
use of Bloom clocks and aims to minimize the amount of data transmitted and the
computational effort required. Bloom clocks expose a risk of false positives,
which means that the comparison of two Bloom Clock Stamps may output a false
result with a certain low probability. To resolve this issue, we introduce the
notion of a Version Digest of the Entity, which must be parameterized to reach
the desired level of confidence. The Version Digest is the root of a Merkle tree
built from the sequence of Events that have been applied to the Entity,
considering the back references forming the DAG and the applicable Join Strategy
in the context.

1. Initial exchange of Bloom Clock Stamps: Each agent sends the other a Bloom
   Clock Stamp and the version Digest representing its current version of the
   Entity.

2. Determination of missing Events: Each agent compares the received Bloom
   Clock Stamp with its own and determines which Events the other agent is
   likely missing based on the causality expressed by the Bloom clocks.

3. Proactive synchronization: Each agent sends the other any missing Events
   that it has in its possession and are likely missing.

4. Check for Bloom Clock false positives: Each agent computes the Version
   Digest of the Entity and compares it to the one received. If they are
   different, the agent enters reactive synchronization mode.

5. Reactive synchronization: In the case where the root of the Merkle tree
   does not match after step 3 or later, the agents enter reactive
   synchronization mode and begin exchanging the data of the Merkle tree in
   order to discover any discrepancies. This can be done efficiently by
   exchanging the hashes of each level of the tree and only requesting the data
   for the branches or leaf nodes that have discrepancies. To begin, each agent
   sends the other the first level under the root hash of their respective
   Merkle trees. They identify the branches that have discrepancies, proceed to
   the next level of the tree, and repeat the process of comparing hashes. When
   they reach a leaf and therefore identify some missing events, they send them
   to the other.
   As the agents exchange the data of the tree, they also update their Bloom
   Clock Stamps and Version Digests to reflect the new information. Once the
   discrepancies have been resolved and the agents' mutual knowledge of the
   Entity is consistent, they exit reactive synchronization mode and continue
   maintaining mutual knowledge as described in step 6.
6. Maintenance of mutual knowledge: After the initial synchronization, the
   agents continue to exchange new Events and update their mutual knowledge
   of the Entity and the corresponding Bloom Clock Stamp as they come. They
   also exchange periodically the Version Hash Digest to ensure that their
   mutual knowledge remains consistent. This is done while the 2 agents are
   online and connected.

## Protocol to resolve collaboratively a fork in a feed and inform the owner of the feed

This protocol is based on the abstract protocol described earlier, with the
following specificities:

* The Entity is a Feed.
* The default Join Strategy is as follows:

    - Keep all branches, sorted by the hash of the first message of the
      branches.
    - If the hashes are equal, use the comparison of the byte strings of the
      full message.
    - In the future, some feeds may be 'typed' and have more specific Join
      Strategies. These Join Strategies must be deterministic and ensure
      that given a set of events (Messages) of a feed, they output the same
      sequence of events.

* The Merkle Tree is built on the sequence of messages in the feed. When there
  is a fork, a sub-Merkle tree is built for each branch of the fork. The root of
  each sub-tree is sorted by the hash of the first message of the branch. A
  Merkle tree is then built on the roots of the sub-trees. The resulting root
  becomes a leaf in the parent Merkle Tree. This process is applied recursively
  if forks occur in branches. This allows for the construction of an efficient
  interactive search protocol for missing messages with a high level of
  confidence, depending on the chosen hash algorithm.

The protocol consists of the following steps:

1. Initial exchange of Bloom Clock Stamps and Version Digests: Each agent
   sends the other a Bloom Clock Stamp and the Version Digest representing its
   current version of the Feed.

2. Determination of missing Messages: Each agent compares the received Bloom
   Clock Stamp with its own and determines which Messages the other agent is
   likely missing based on the causality expressed by the Bloom clocks.

3. Proactive synchronization: Each agent sends the other any missing Messages
   that it has in its possession and are likely missing.

4. Check for Bloom Clock false positives: Each agent computes the Version
   Digest of the Feed and compares it to the one received. If they are
   different, the agents enter Reactive synchronization mode.

5. Reactive synchronization: The agents exchange the data of the Merkle tree
   in order to discover any discrepancies. This can be done efficiently by
   exchanging the hashes of each level of the tree and only requesting the data
   for the branches or leaf nodes that have discrepancies. When they reach the
   leaf nodes and identify missing Messages, they send them to the other agent.
   As the agents exchange the data of the tree, they also update their Bloom
   Clock Stamps and Version Digests to reflect the new information. Once the
   discrepancies have been resolved and the agents' mutual knowledge of the Feed
   is consistent, they exit reactive synchronization mode and continue
   maintaining mutual knowledge as described in step 6.

6. Maintenance of mutual knowledge: After the initial synchronization, the
   agents continue to exchange new Messages and update their mutual knowledge of
   the Feed and the corresponding Bloom Clock Stamp as they come. They also
   exchange the Version Hash Digest periodically to ensure that their mutual
   knowledge remains consistent.

### Open issues

1. Informing other peers and the owner of a fork. <br/>
   There are currently two options for informing other peers and the owner of
   a fork in a feed:

    1. Peers can publish a "negative vote" from the current SSB protocol on
       the first message of the branch to be abandoned, with a specific reason
       given. This message should also include proof in the form of a copy of
       the first message of the selected branch with the signature, to show that
       the owner of the feed effectively created the fork. Other peers and the
       owner can then take appropriate measures to stop propagating the bad
       branch of the feed.
    2. Peers can publish a new message type to inform other peers of the fork
       and the result of the join. What should it contains ? What should be
       the expected behavior of the other peers ?

2. It may be useful to add some 'join' message type that allows the feed
   owner to inform other peers of the result of the join and reduce the
   complexity of the DAG of the feed.

## Protocol to efficiently share status and synchronize feeds shared by 2 connected peers

The protocol for efficiently sharing status and synchronizing feeds shared by
two connected peers is based on the use of Bloom clocks and aims to minimize the
amount of data transmitted and the computational effort required.

In the initial exchange of feed filters, each peer sends the other a standard
bloom filter (or cuckoo filter) containing the IDs of the feeds that it is
replicating. This allows the peers to quickly determine, with high probability,
the list of feeds that they have in common by computing the intersection of
their filters.

The Entity in this protocol is the set of common feeds, each of which can be
considered as a branch of a fork at an empty root. The Join Strategy for these
feeds is two-fold: at the first level, the feeds are all kept and sorted by
comparing the byte string of the feed ID. At the feed level, the Join Strategy
applicable to that feed, as described in a previous section, is applied. The
Merkle Tree for these feeds is the root of a Merkle Tree whose leaves are the
roots of the Feed Merkle Trees, also as defined in the previous section.

In the event of discrepancies, the "reactive synchronization" process becomes
more complex. The peers must first determine if the discrepancy is due to a
false positive in the initial establishment of the common set of feeds using a
bloom filter (or cuckoo filter). If a false positive is detected, the list of
common feeds is corrected accordingly and the process returns to step 2. If the
discrepancy is determined to be a true issue, the protocol "Protocol to resolve
collaboratively a fork in a feed and inform the owner of the feed" is applied to
the feed detected as being truly common but discrepant.

1. Initial exchange of feed filters: Each peer sends the other a standard bloom
   filter (or cuckoo filter) containing the IDs of the feeds that it is
   replicating. This allows the peers to quickly determine, with high
   probability,
   the list of feeds that they have in common by computing the intersection of
   their filters. This also obfuscates the list of feeds that one peer is
   interested in, as the other peer can only determine with some probability the
   list of feeds followed by the peer even if it is aware of all existing feed
   IDs
   in the SSB network.

2. Determination of common feeds: Each peer computes the intersection of the
   received feed filter with its own filter and determines the probable list of
   feeds that they have in common.

3. Merging of common feeds: The Entity of the abstract protocol if the set
   of common feeds, each of which can be considered as a branch of a fork at an
   empty root. The "join" of these feeds can be represented by a single
   Bloom clock stamp sum of the Bloom clock stamps of the common feeds. The
   second message of the protocol is the peer sending each other
   the "merged" Bloom clock stamp for the whole set of feeds that they have
   presumably in common.

4. Proactive synchronization: Each peer receives the "merged" Bloom clock stamp
   from the other peer and can quickly scan its own database to determine which
   messages in this common list of feeds the other peer is likely missing. It
   then
   sends these missing messages to the other peer immediately. If a peer
   receives a
   message about a feed that it does not replicate, this is due to a false
   positive
   due to the probabilistic nature of the bloom filter used in step 1. The peer
   informs the other peer of this, they updates their list of common feeds
   and step back to step 3.

5. Reactive synchronization: After step 4, if the peers detects a
   discrepandcy of their Merkle trees, they must first determine
   if the discrepancy is due to a false positive in the common feed list
   established in step 2. If this is the case, the peer corrects its list of
   common
   feeds and goes back to step 3. If the discrepancy is not due to a false
   positive, the peer follows the protocol described in the following section "
   Protocol to resolve collaboratively a fork
   in a feed and inform the owner of the feed" for the feed detected as truly
   common but discrepant.

6. Maintaining session Bloom clock: Subsequently, as the connected peers receive
   updates from other peers in the network, they must maintain a Bloom clock
   and Merkle Trees for the session. When they receive a message from another
   third peer that is likely missing for the other peer, they proactively
   transmit it to the peer and increment the session Bloom clock accordingly.
   This ensures that the connected peers are able to efficiently synchronize
   with each other and stay up to date with each others and the rest of the
   network.

## Protocol to efficiently synchronize threads like structure in SSB

SSB-tangle is a library for working with causally ordered messages, known as "
threads," in the Secure Scuttlebutt (SSB) network. It allows for the creation of
threads by linking messages together through "backlinks," which allows for the
construction of a graph of the relative timing of messages. SSB-tangle also
defines strategies for combining a series of messages and determining the
validity of merging messages.

One potential criticism of SSB-tangle is the inefficiency of syncing knowledge
about a specific thread between peers. The process of querying each individual
message from peers can be time-consuming and resource-intensive. It may be
beneficial to develop a more efficient protocol for syncing knowledge about
threads between SSB peers, possibly utilizing Bloom Clocks and Merkle trees.

Another potential limitation of ssb-tangle is that it relies on backlinks to
represent the causal ordering of messages in a thread. This means that if a
message is not received by a peer, it will not be included in the thread and may
cause gaps in the thread's structure. Additionally, if a message is received out
of order (e.g. a reply to a message that has not yet been received), it may not
be properly integrated into the thread. This can lead to inconsistencies in the
thread's structure and make it difficult for peers to accurately sync and
display their knowledge of the thread.

There is a large similarity between a thread-like structure in SSB and a feed
that forks. Messages in a feed have a backlink to the previous message in the
feed in the "previous" field, while messages in a thread in SSB have a backlink
to the previous message(s) in the thread in the "branch" field.

In our abstract model, a thread is the Entity and the messages, regardless of
the feed they come from, are the Events updating that Entity. When two agents
create a message with the same value in the "branch" field, this is a fork in
the thread. And when a message backlinks to more than one message, this
corresponds to a Join event.

Based on this, we can derive an efficient protocol to sync a thread-like
structure in SSB between two agents. This protocol is based on the abstract
protocol described earlier and is very similar to the protocol to resolve
collaboratively a fork in a feed and inform the owner of the feed. The main
difference is that the Entity is a thread instead of a Feed.

The protocol to synchronize a set of Thread between to agent is very similar to
the protocol "Protocol to efficiently share status and synchronize feeds shared
by 2 connected peers"

1. Initial exchange of Thead set filters: Each peer sends the other a standard
   bloom filter (or cuckoo filter) containing the root message id of the
   threads that it is interested in. What are the threads of interest for some
   peer is application dependent.

2. Determination of common threads: Each peer computes the intersection of the
   received thread filter with its own filter and determines the probable
   list of threads for which they have a common interest.

3. Merging of common threads: The Entity of the abstract protocol if the set
   of common threads, each of which can be considered as a branch of a fork
   at an empty root. The "join" of these Threads can be represented by a single
   Bloom clock stamp sum of the Bloom clock stamps of each Threads.

4. Proactive synchronization: Each peer receives the "merged" Bloom clock stamp
   from the other peer and can quickly scan its own database to determine which
   messages in this common list of threads the other peer is likely missing. It
   then sends these missing messages to the other peer immediately. If a peer
   receives a message about a thread that it does not replicate, this is due
   to a false positive due to the probabilistic nature of the bloom filter used
   in step 1.
   The peer informs the other peer of this, they update their list of common
   threads and step back to step 3.

5. Reactive synchronization: To build the root Merkle tree, the peer have to
   maintain a Merkle tree per session whose leaf are the root of the Merkle
   tree of the Thread which are build and maintain similarly to feed merkle
   trees.

6. Maintaining session Bloom clock and Merkle tree as new messages arrives.

## Protocol for establishing a consensus on elapsed time

### Use cases

There are several potential use cases for a consensus on elapsed time in a
distributed network like Secure Scuttlebutt:

1. Timestamps: The global counter could be used as a timestamp for events in
   the network, allowing for the ordering of events and the calculation of time
   intervals between them.

2. Scheduling: The global counter could be used to schedule events in the
   network, ensuring that they occur at a specific time relative to the counter.

3. Time-based events: The global counter could be used to trigger events in
   the network that are based on elapsed time, such as expiration of a resource
   or the execution of a task.

4. Network synchronization: The global counter could be used to synchronize
   the clocks of different nodes in the network, ensuring that they are all
   using the same reference time.

5. Proof of elapsed time: The global counter could be used to provide a
   non-interactive proof of elapsed time, which could be used to enforce waiting
   periods or expiration dates for resources.

### Principles

A possible approach to establishing a consensus on elapsed time could involve
using a combination of cryptographic techniques, such as digital signatures and
hash chains, to ensure that each increment to the global counter is valid and
that sufficient time has elapsed between increments.

One way to implement such a protocol could involve the following steps:

1. Each peer have a unique key pair for signing their increments to the global
   counter.
2. When a peer wants to increment the global counter, they must first prove that
   they have waited a certain amount of time since the last increment. This can
   be
   done by creating a hash chain, where each element in the chain is the hash of
   the previous element concatenated with a timestamp. The length of the hash
   chain
   must be sufficient to demonstrate that a sufficient amount of time has
   elapsed.
3. Once the hash chain has been created, the peer can sign the final hash in the
   chain with their private key to create a digital signature. This signature
   can
   be used to prove that the peer has indeed waited the required amount of time.
4. The peer can then broadcast their increment to the global counter, along with
   the digital signature and the hash chain, to the rest of the network.
5. Other peers can verify the validity of the increment by checking the digital
   signature, reconstructing the hash chain, and ensuring that the final hash in
   the chain matches the signed hash and that sufficient time was spent between
   timestamps of the message of the chain. If the increment is valid, the peer
   can
   update their copy of the global counter and broadcast their own increment to
   the
   network.

By considering the chain of increments to the global counter as a thread, it is
possible to efficiently synchronize these increments between peers using the
protocol previously described. This allows multiple peers to increment the
global counter simultaneously, and to merge the resulting increments into
multiple branches of the thread. These branches provide some guarantees that the
increments will eventually occur, and will be mostly continuous, even in a
network like SSB where peers may not always be connected and messages are
replicated to all nodes.

## Rules

A few key aspects of the core SSB protocol that are relevant to this protocol:

* Peers are not allowed to propagate messages with timestamps in the future
* Timestamps of messages in a feed must be monotonically increasing
* Replication of feeds occurs in two circles: first circle is the feeds that
  a peer follows, second circle is the feeds that the first circle follows
* A feed is guaranteed to have a full and secured knowledge of the followed
  feeds by the feeds that it follows (first circle)
* It is possible to have a good knowledge of the followers of the feeds in
  the first circle and second circle using a previously described protocol,
  though this knowledge is not guaranteed to be complete from a security
  perspective.

The counter is a Bloom clock whose the hashe functions are deriving the
indexes from the feed id of the incrementer.

The counter is equal to the max value across the elements of the Bloom clock.

The protocol requires 3 messages:

1. 'Initiate' message: this message creates a global counter and specifies the
   parameters: hash functions and the size of the Bloom Clock, the maximum
   range of the Bloom Clock, minimum delay between increments and ratio of
   followers designated as incrementer.
2. 'Participate' message: this message is the commitment of one feed to
   participate in the protocol (or stop participating).
3. 'Increment' message: this message publish an increment.

All these messages are Thread-like messages. The "root" is "Initiate" message.
The branches in the "branch" field, in 'Participate' and 'Increment' contains
the list of Increment messages that the feed is aware of at the time it
published its message.

### Rule 1: which peer increment the counter ?

In order to establish a consensus on elapsed time, the following rules should be
implemented:

* Only certain peers, determined by the SELECT function, are allowed to
  increment the global counter. These peers must have previously committed to
  participating in the protocol, and must be part of the first circle of the
  feed publishing the increment message.
* The SELECT function is computed as follows:

    * Both the hash of the increment message and the feed ID of the followers
      are treated as 256-bit integers.
    * For each follower, calculate (Message Hash * Feed ID) % R.
    * If the result is 0 for any followers, they are designated as the next
      incrementers.

* The maximum range of the Bloom Clock must not be exceeded when incrementing
  the counter. This prevents a small group of peers from incrementing the
  counter in isolation.
* Peers in the first circle must only increment the counter from an increment
  message published by a peer in their first circle, as determined by the SELECT
  function.
* Incrementers must not publish an increment with a message timestamp that is
  earlier than (timestamp of previous increment + T seconds).
* R and T are key parameters of the protocol. R, the ratio of followers
  designated as incrementers, influences the speed at which the global counter
  is propagated, while T, the minimum delay between increments, affects the
  number of messages created on the network. A good balance between these
  parameters should be chosen based on the needs of the application using the
  counter.
* Optional: To increase the security of the proof of delay, incrementers may
  be required to perform a crypto delay function and publish a non-interactive
  Proof of Work in the 'increment' message.

### Rule 2: which peer verify the increments and what are the consequences of a failed verification ?

* All peers participating in the protocol must verify the increment messages
  of the first and second circles of the feeds they follow.
* The strict minimum timestamp for an increment message is equal to the
  timestamp of the 'Initiate' message plus the number of increments multiplied
  by the delay per increment.
* All increment messages must be constructed by joining the Bloom Clock stamp
  of all the increments referred to in the 'branch' field and adding the feed ID
  of the incrementer.
* Any feed that publishes an increment that does not adhere to Rule 1 must be
  blocked definitively by all participants.
* When a peer sees an 'increment' message, it must temporarily block the
  propagation of messages from the designated incrementer in its first circle
  with a larger timestamp than the minimum timestamp of the next increment.
* The minimum timestamp is equal to all the timestamps of the counter at that
  level that the peer is aware of. Participants in the protocol should maintain
  at least one chain of increments from the root, using the previous
  synchronization protocol for threads with some modifications. When merging
  multiple branches, they should keep only the branches that contain increments
  with their own first and second circles, as well as the branch that results in
  the minimum timestamp.
* Participants may also choose to synchronize on a range of Block Clock
  stamps from the top, rather than the full range to the root. The exact range
  will depend on the needs of the application using the global counter. Some may
  require full range synchronization to the root, while others may only require
  a certain range in the past.
