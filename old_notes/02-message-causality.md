# Messages causality

The Bloom clock [@abs-1905-13064][@Kshemkalyani2020TheBC][@Misra2020TheBC] concept is very 
simple.  It is based on a 
counting bloom filter data structure. 

Let's consider an entity X that can be modified concurrently by several 
agents of distributed system.  When one agent modify this entity, it emits a 
message with a reference to the entity and the bloom clock corresponding.

When an agent receive a modification from another agent, it can update his 
knowledge of X and its version of the bloom clock.

As it is based on a counting bloom filter, there is computable probability 
of false response to some queries.  

There are 2 parameters defined by the protocol that must be known by all 
parties.  Those parameter can be application dependent and has some optimal 
values dependent on the application and the chosen tradeoff between the 
byte size of the clock, the computational effort to process the clock and the
probability of false response to some queries.  We will examine in 
subsequent section those queries and the impact of false response.

## Base rules of update of the bloom clock
The Bloom Clock (BC) is counting bloom filter of size N with a Random Oracle 
function (ROF)
function that produce k indices between 0..(N-1) from the update message.

A update message (UM) is a message in some feed that "update" a some 
identifiable 
SSB entity (IE)(a message, a feed-id or a blob). 

!!! example

    For instance, a message M1 of type "post" that refers to another message 
    of type "post" M0 with the field "root" is an update message of that 
    message M0.  Similarly, a message of type "vote" is an update message.

**_Rule 1_**: The initial Bloom clock of any identifiable SSB entity is all 0. The 
default parameters N and the RO and how they can be modified in some cases are 
specified in the section [default parameters](default-parameters.md).

**_Rule 2_**: When an agent produces an update message UM on some identifiable 
entity IE with a current version BC(n), it produces a BC(n+1) by 
incrementing the k counters at the indices produced by the random oracle 
function ROF(UM). The UM is added to the agent feed together with BC(n+1) 
and the reference to the identifiable entity IE

**_Rule 3_**: When an agent receive an update message UM with the Bloom clock 
BC(x) on some identifiable entity IE with a current version BC(n), if update its
knowledge about the IE and update the version of IE in its db to BC(n+1) = 
[max(BC(n)[i], BC(x)[i]) for i in 0..(N-1)]

!!! note

    The question here would be to decide if this protocol is too be used for 
    each specific type of messages or if it make sense to implement at the 
    enveloppe level agnostically to the type of message.  Both as pro and con.
    But given (1) that feeds will probably tends to specialize per 
    application with metafeed and (2) adding it at envelop level do not 
    break any backward compatibility and can be just considered as new 
    capability that 2 agents may use to improve their exchanges after 
    negociation, (3) that make a directly usable primitive for any new 
    protocol designed on SSB with a single implementation, I think it makes 
    sense to implement at envelop level, while allowing to reuse at deeper 
    level for specific applications (or hiding in encrypted boxes).

## Interesting properties

Partial ordering

:   If all elements of BC~a~ are pairwise smaller or equal to all elements 
    of BC~b~, event A precedes event B.  
    Assuming that bloom filter increments are random, one can immediately see 
    that the larger the "distance" between  BC~a~ and BC~b~, the larger the 
    probability of false positive comparison.  
    The probability of false positive comparison under the assumption of 
    the Random Oracle Function is (1 - (1 - 1/n)^|BC~b~|^)^|BC~a~|^ with 
    |BC~x~| = sum of elements of BC~x~
    So, that probability depends only on the size n of the bloom clock, the 
    parameter k and the number of events between the event A and the event B.

Common root high bound

:   Let consider BC~a~ and BC~b~ so that neither precedes one according to 
    the previous definition.  Event A and Event B are part of 2 branches of 
    a sequence of event and there exists an event C, the root of both 
    branches so that it is the largest BC~c~ that precedes both BC~a~ and BC~b~.
    In other words, the hypothetical event X is with BC~x~ = element-wise 
    min of BC~a~and BC~b~ is the lasgest possible common root of the branch 
    ending at A and the branch ending at B.
    Consequently, an agent having the chain of events ending at event A and 
    only the clock BC~b~ of event B and determine with a high probability the 
    past event in the chain that is the fork point at which the branch ending 
    at event B started.

## Simple example
For this example, we consider that Alice, Bob and Carol replicates each
others and that messages are simple "post" and replies to an initial "post"
from Alice .

To make things more clear, lets take a simplified example:  

1. Alice creates a message A0.
2. Bob receives the message A0 and answers with the messages B1
3. Carol receives the message A0 and answers with the messages C1
4. Bob receives the message C1 and answer with the message B2
5. Alice receives B1
6. Alice receives B2
6. Carol receives B1 
7. Carol receives B2
8. Alice receives C1

We will use a counting bloom filter of size 4 and with 2 hash functions for 
the example.  Let's examine the knowledge of each of the parties at every step.

``` mermaid
flowchart BT
    classDef green fill:#0f6
    classDef red fill:#bb0
    subgraph Alice
        A0[A0 - 0,0,0,0] -.->|step 1| AI[ ]
        AB2[B2 - 1,0,2,2]:::green -.-> AB1
        AC1[C1 - 0,0,1,1] -..-> A0
        AB1[B1 - 1,0,1,0]:::green -.-> A0
        AB2 -.-> AC1
        TA[Thread = A0,B1,C1,B2]:::red --x AB2
    end
    subgraph Carol
        CA0[A0 - 0,0,0,0]:::green ==o|step 3| A0     
        C1[C1 - 0,0,1,1] -.-> CA0
        CB1[B1 - 1,0,1,1]:::green -..-> C1
        CB2[B2 - 1,0,2,2]:::green -..-> CB1
        
        AC1 =====o|step 9| C1
        TC[Thread = A0,C1,B1,B2]:::red --x CB2
    end
    subgraph Bob
        BA0[A0 - 0,0,0,0]:::green ==o|step 2| A0             
        B1[B1 - 1,0,1,0] -.-> BA0
        BC1[C1 - 1,0,1,1]:::green -.-> B1
        BC1 ===o|step 4| C1  
        B2[B2 - 1,0,2,2] -.-> BC1
        AB1 ==o|step 5| B1
        AB2 ==o|step 6| B2
        CB1 ====o|step 7| B1
        CB2 ====o|step 8| B2
        TB[Thread = A0,B1,C1,B2]:::red --x B2
    end 
```


``` mermaid
%%{init: { 'gitGraph': {'mainBranchName': 'Alice'}} }%%

gitGraph
   commit id: "A0 [0,0,0,0]" tag: "step 1"
   branch Bob
   commit id: "A0 [0,0,0,0] " type: HIGHLIGHT
   commit id: "B1 [1,0,1,0]" tag: "step 2"
   checkout Alice
   branch Carol
   commit id: "A0 [0,0,0,0]  " type: HIGHLIGHT
   commit id: "C1 [0,0,1,1]" tag: "step 3"
   checkout Bob
   merge Carol id: "C1 [1,0,1,1] " type: HIGHLIGHT
   commit id: "B2 [1,0,2,2]" tag: "step 4"
   checkout Alice
   merge Bob id: "B1 [1,0,1,0] " tag: "step 5" type: HIGHLIGHT
   merge Carol id: "C1 [1,0,1,1]  " tag: "step 9" type: HIGHLIGHT
   commit id: "B2 [1,0,2,2] " tag: "step 6" type: HIGHLIGHT
   
   checkout Carol
   merge Bob id: "B1 [1,0,1,1]  " tag: "step 7" type: HIGHLIGHT
   commit id: "B2 [1,0,2,2]  " tag: "step 8" type: HIGHLIGHT
   checkout Alice
```

At step 1, Alice publish the initial post A0.  The version of A0 in its db 
is [0,0,0,0] and others are not yet aware of that post.

At step 2, Bob receives A0.  So, internally it initiates the version to [0,0,
0,0]. It generates a reply B1, apply the random oracle function ROF(B1) = [0,
2] and therefore increment the version of A0 in its db to [1,0,1,0].  In its 
feed, it adds the message B1 with the clock [1,0,1,0] and the link to A0

At step 3, Carol receives A0.  So, internally it initiates the version to [0,0,
0,0]. It generates a reply C1, apply the random oracle function ROF(B1) = [2,
3] and therefore increment the version of A0 in its db to [0,0,1,1].  In its
feed, it adds the message C1 with the clock [0,0,1,1] and the link to A0.

At step 4, Bob receives the message C1[0,0,1,1] from Carol and the current 
version of A0 in its db is [1,0,1,0] and is the result of [A0,B1].  It 
updates its DB so that A0 is now the result of [A0,B1,C1] and has the 
version [1,0,1,1]. It generates a reply B2, apply the random oracle function 
ROF(B2) = [3,4] and therefore increment the version of A0 in its db to [1,0,
2,2] and is the result of [A0,B1,C1,B2].  In its feed, it adds the message B2 
with the clock [1,0,2,2] and the link to A0.

At step 5, Alice receives B1[1,0,1,0] from Bob and the current
version of A0 in its db is [0,0,0,0]. It updates its DB so that A0 is now the
result of [A0,B1] and has the version [1,0,1,0].

At step 6, Alice receives B2[1,0,2,2] from Bob and the current
version of A0 in its db is [1,0,1,0] and the result of [A0,B1].  Because the 
gap (3) between its current version A0 and the message B2 is larger than k (2), 
Alice knows immediately that Bob is aware of one more update messages that 
Alice doesn't know.  Alice can then query Bob (or any other peers) for all update 
messages about A0 between [1,0,1,0] and [1,0,2,2] and insert the result in 
its update sequence of A0 [A0,B1,x,B2]. In the meantime, in its db A0 is 
[A0,B1,B2] with the version [1,0,2,2].

If Alice would have queried Bob, Bob would be able to immediately respond 
with C1 because it knows both points in the range [1,0,1,0] and [1,0,2,2]

``` mermaid
flowchart LR
    classDef green fill:#0f6
    classDef red fill:#bb0
    subgraph Bob
        direction RL
        BA0[A0 - 0,0,0,0]              
        B1[B1 - 1,0,1,0] -.-> BA0
        BC1[C1 - 0,0,1,1]:::green -.-> BA0
        B2[B2 - 1,0,2,2] -.-> BC1 & B1
        TB[Thread = A0,B1,C1,B2]:::red --x B2
    end 
```

But, in our example, Alice queries Carol for range [1,0,1,0] and [1,0,2,2].  At 
that step, Carol knows A0 as being the result of [A0,C1] with version [0,0,1,1].

``` mermaid
flowchart LR
    classDef green fill:#0f6
    classDef red fill:#bb0
    subgraph Carol
        direction RL
        CA0[A0 - 0,0,0,0]
        C1[C1 - 0,0,1,1] -.-> CA0
        TC[Thread = A0,C1]:::red --x C1
    end 
    subgraph Alice
        direction RL
        AA0[A0 - 0,0,0,0]              
        AB1[B1 - 1,0,1,0] -.-> AA0
        Ax[?]:::green -.-> AA0               
        AB2[B2 - 1,0,2,2] -.-> Ax & AB1
        TA[Thread = A0,B1,C1,B2]:::red --x AB2
    end 
```

Because the low bound of the range [1,0,1,0] is not included in its version of 
A0, Carol cannot answer to Alice.  But, Carol knows that Alice knows some 
branch of updates that it don't know.  And Carol can determine that its largest 
common version under that bound is [0,0,0,0], so instead of returning some 
messages to Alice, it will ask Alice to send the messages it has between [0,0,
0,0] and [1,0,2,2].  

``` mermaid
flowchart LR
    classDef green fill:#0f6
    classDef red fill:#bb0
    subgraph Carol
        direction RL
        CA0[A0 - 0,0,0,0]
        C1[C1 - 0,0,1,1] -.-> CA0
        CB1[B1 - 1,0,1,0]:::green -.-> CA0
        CB2[B2 - 1,0,2,2]:::green -.-> CB1 & C1
        TC[Thread = A0,C1,B1,B2]:::red --x CB2
    end 
```

Alice will then send to Carol B1[1,0,1,0] and B2[1,0,2,2] and as 
Carol receives it, its version of A0 becomes [1,0,2,2], the result of [A0,C1,
B1, B2] and Carol can now determine than what Alice is missing is C1[0,0,1,1]
and send it to Alice.  Alice gets finally C1 and after updating its own view 
of A0 has a version [1,0,2,2].  

``` mermaid
flowchart LR
    classDef green fill:#0f6
    classDef red fill:#bb0
    subgraph Alice
        direction RL
        AA0[A0 - 0,0,0,0]              
        AB1[B1 - 1,0,1,0] -.-> AA0
        Ax[C1 - 0,0,1,1]:::green -.-> AA0               
        AB2[B2 - 1,0,2,2] -.-> Ax & AB1
        TA[Thread = A0,B1,C1,B2]:::red --x AB2
    end 
```

So Alice can be convinced that he has learned all what was missing that it 
detected when it received B2 and Carol at the same time updated his knowledge
up to version [1,0,2,2].



!!! note 
   
    The Bloom Clock provides a partial ordering of the messages.  To have a 
    complete ordering and disambiguates between branches, some additionnal 
    application specific strategy can should be applied. Depending on the 
    application, it can be the UI or the application protocol that specifies it.

!!! note

    One may find the protocol between Alice and Carol complex.  Remember 
    that this is a toy example.  The number rounds (4) is fixed and independent 
    of the number of messages missing in both Alice and Carol.  Compare it 
    to the current ssb-tangle algorithm that requires each Alice and Carol 
    to query one by one missing messages to rebuild the complete graph. 





