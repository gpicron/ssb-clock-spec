# Overview

## Context

In SSB, no agent is supposed to have a complete replica of all the SSB 
community feed.  One agent replicates the feeds that it follows and the 
feeds declared by those feeds to be followed.  Moreover, at any point in 
time, one agent cannot be sure to have a complete replica of the feeds that 
it follows or that a feed that he follows has not forked. 

In the context of message of type `post`, the fields `root` and `branches` may 
be used to express some causality in a series of messages in reply to an initial
root message.  When one agent post a reply to some message, it can express 
the merging of several branches it is aware of at the time of the post by 
filling the `branches` field.  In order to "merge" the branches, the agent 
must either rely on the timestamp provided by the author or by the reception 
time.  In both case, the choice of order of the messages is solely a UI 
decision and cannot be resolved by the protocol.

This is somewhat acceptable for a forum-like application giving a trade-off 
between complexity and security, but it may be unacceptable for applications 
like ssb-token or multiparty games.

## Problem statement 1: feed forks
At best, an agent can detect at some point a fork of a feed he is following but 
his unable to determine which branch should be followed. 
A fork of a feed is either 

- unintentional due to some bug or mis-manipulation by the user
- intentional due to some adverse agent

In practice, for unintentional case, it often implies that the user 
continuing to post messages on the second branch will not notice immediately 
that his messages are not propagated anymore.  Other agents will recognize 
that his messages are not part of the first branches and will discard them 
silently.

For the intentional case, it permits to an adverse agent to tell a different 
"story" to 2 parts of the community.  There is no mean for 2 agents, one in 
each part, to agree on the story from the adverse agent.  If one distribute 
messages of the adverse agent to the other, the other will just think it is a 
corrupted stream of message and discard them silently.

## Problem statement 2: rebuilding the graph of causality

Based on the feed that it replicates, an agent analysing a message can 
determine the root message of the graph and the messages just preceding 
according to the author of that messages.  Meanwhile, if that root message 
of one of those preceding are not part of the feeds that it replicates or if 
it has not yet received the corresponding updates, it is unable to 
reconstitute the graph.

If it has several messages referring to the same root, it can determine that 
they are in the same graphes but is unable to determine if there is some 
causality between them without rebuilding the full graph.

In other to rebuild the full graph, it must query around the connected peers 
for the missing messages.  And, once it received it, if it wishes to be 
confident that the SSB protocol was respected by the author, it must replicate 
its entire feed.

## Proposed solution(s)
Several algorithm have been designed, especially for trusted environment, to 
allow an agent of a distributed system to determine the causality between 2 
events without having to rebuild the full graph of events (Lamport clocks, etc.)

While not specifically designed for untrusted environment, the Bloom clock 
[@abs-1905-13064] offer some potentialities in the context of SSB. 

In the following sections, we make 3 proposals to enhance the 
current SSB protocol with regard to the problems mentioned here before.

1. Using a Bloom clock to reflect the causality between messages issued in 
  different feeds in response to a given message.
2. A protocol to establish a consensus of the elapsed time since some event.
3. A protocol to resolve collaboratively a fork in a feed and inform the 
   owner of the feed.



