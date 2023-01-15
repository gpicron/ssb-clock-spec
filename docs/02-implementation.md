# Implementation considerations

The "Implementation considerations" section of this design document discusses
the practical considerations that need to be taken into account when
implementing the protocols outlined in the previous sections.

## Core data structures and algorithms

### Bloom clock implementation

The core data structure used in the protocols outlined in this design document
is the Bloom clock. The Bloom clock is a counting bloom filter data structure
that allows for the expression of causality between messages within a feed. It
consists of a counter array that is modified by the addition of elements, or 
hashes, representing events. The Bloom clock has two parameters, N and k, which
determine its size and the number of indices produced by the Random Oracle
Function (ROF), respectively.

The base rules for updating the Bloom clock are as follows:

* When an agent produces an update message (UM) on some identifiable entity (IE)
  with a current version BC(n), it produces a BC(n+1) by incrementing the k
  counters at the indices produced by the ROF(UM). The UM is added to the
  agent's
  feed along with BC(n+1) and the reference to the identifiable entity IE.
* When an agent receives an update message UM with the Bloom clock BC(x) on some
  identifiable entity IE with a current version BC(n), it updates its knowledge
  about the IE and updates the version of the IE in its database to BC(n+1) = [
  max(BC(n)[i], BC(x)[i]) for i in 0..(N-1)].
* The Bloom clock is useful for expressing the causality between events, or
  messages, in a distributed system. It is independent of the order of branches
  when merged, and supports efficient removal of events by keeping track of the
  hashes that were added to the Bloom clock. The Bloom clock can also be used to
  estimate the number of events that occurred between two events in a chain, and
  is encoded in a compact format to allow for efficient storage and
  transmission.

The BloomClockStamp is encoded in a compact format to allow for efficient storage
and transmission, and can support different block sizes and formats to allow for
flexibility in the trade-off between size and computational effort.

The model of a BloomClockStamp is as follows:

* The BloomClockStamp is a counter array of length N, where N is a parameter 
  that determines the size of the BloomClockStamp.
* The BloomClockStamp is initialized to all zeros.
* When an event occurs, a hash is added to the BloomClockStamp by incrementing 
  the k indices produced by the Random Oracle Function (ROF) by 1.
* The BloomClockStamps can be joined by setting each counter of the array to 
  the maximum of the two values.
* The BloomClockStamp can be serialized into a compact byte representation for
  efficient storage and transmission.

The encoding of a BloomClockStamp is as follows:

* The minimum value of the elements of the array is the base and is encoded 
  using a varint.
* The increment from the base of elements of the vector of the vector are 
  encoded using bitpacking.
* There are 3 parameters:

  * The bitpacked chunk size CZ and the number of chunks C such that the C * 
    CZ = N the size of the BloomClockStamp
  * The bitpacking format which determine the number of bits per chunk used 
    to specify the format of the chunk and the format of the chunks used.
    Consequently, it also inmplies the maximum range encodable between the 
    min and max elements of the array.  For instance, 
    * the format (1,2,4,8) use 2 bits per chunk to specify format of the 
      chunk and 1, 2, 4 or 8 bits to encode the elements of the chunk. The 
      max range is 256.
    * the format (0,1,2,8) use 2 bits per chunk to specify format of the
      chunk and 0, 1, 2 or 8 bits to encode the elements of the chunk. It 
      can be interesting for very large Bloom Clock with a small number of k 
      of hashes functions are it implies that most of the chunk will be all 
      zeros or very small.
    * the (0,1,2,4,8,16,24,32), will use 3 bits per chunk and allows to 
      encode a range of 2^32.
    * other formats may be invented on the same logic.
  * Note: the chunk size must be a multiple of 32 so that the chunk 

So, at byte level, it is encoded as:
- 1 positive varint : the base
- A array of CZ n-bit integers representing the format of each chunks.
- 0-Padding at 32 bit
- Element Chunks encoded according the format.

There are 3 functions on BloomClockStamp:

#### compare(BCS1,BCS2)

The compare function takes two BloomClockStamps as input and returns one of
four values:

* BCS_PRECEDES: If all elements of the first BloomClockStamp (BCS1) are
pairwise smaller or equal to all elements of the second BloomClockStamp
(BCS2), then it can be inferred that BCS1 occurred before BCS2.
* BCS_SUCCEEDS: If all elements of BCS2 are pairwise smaller or equal to
all elements of BCS1, then it can be inferred that BCS2 occurred before BCS1.
* BCS_EQUALS: If all elements of BCS1 are equal to all elements of BCS2,
then it can be inferred that BCS1 and BCS2 occurred at the same time.
* BCS_FORKED: If none of the above conditions are met, then it can be
inferred that BCS1 and BCS2 are in separate branches after a fork.

```` typescript
function compare(BCS1: BloomClockStamp, BCS2: BloomClockStamp): ComparisonResult {
  if (BCS1.length !== BCS2.length) {
    throw new Error("Cannot compare BloomClockStamps of different lengths");
  }

  let equal = true;
  let BCS1Precedes = false;
  let BCS2Precedes = false;
  for (let i = 0; i < BCS1.length; i++) {
    if (BCS1[i] < BCS2[i]) {
      BCS1Precedes = true;
      equal = false;
    } else if (BCS1[i] > BCS2[i]) {
      BCS2Precedes = true;
      equal = false;
    }
  }

  if (equal) {
    return ComparisonResult.Equal;
  } else if (BCS1Precedes && BCS2Precedes) {
    return ComparisonResult.Forked;
  } else if (BCS1Precedes) {
    return ComparisonResult.BCS1Precedes;
  } else {
    return ComparisonResult.BCS2Precedes;
  }
}

enum ComparisonResult {
  BCS1Precedes,
  BCS2Precedes,
  Equal,
  Forked
}

````

#### increment(BCS, hash, ROF)

The increment function takes a BloomClockStamp (BCS) and a hash as input,
and returns a new BloomClockStamp that is the result of incrementing the k
counters at the indices produced by the ROF(hash).

```` typescript
function increment(BCS: BloomClockStamp, hash: Hash, ROF: RandomOracleFunction): BloomClockStamp {
  let indices = ROF(hash)  // compute indices to increment using the ROF function
  for (let i of indices) {
    BCS[i]++  // increment the counters at the computed indices
  }
  return BCS  // return the updated BloomClockStamp
}
````

#### join(BCS1, BCS2)

The join function takes two BloomClockStamps (BCS1 and BCS2) as input, and
returns a new BloomClockStamp that is the result of setting each counter of
the array to the maximum of the two values.

```` typescript
function join(BCS1, BCS2) {
  // Initialize result BloomClockStamp to the same size as BCS1
  let result = new Array(BCS1.length)
  
  // Set each element of result to the maximum of the corresponding elements of BCS1 and BCS2
  for (let i = 0; i < BCS1.length; i++) {
    result[i] = Math.max(BCS1[i], BCS2[i])
  }
  
  // Return the result
  return result
}
````

### The DAG

A DAG (Directed Acyclic Graph) is a data structure used to represent the
causality between events in a distributed system. It consists of a set of
nodes, where each node represents an event and contains a reference to the
previous node in the chain. The DAG is a directed graph, meaning that the
edges connecting the nodes have a direction, and it is acyclic, meaning that
there are no cycles in the graph.

The model of a DAG is as follows:

* The DAG consists of a set of nodes, each node representing an event and
containing a reference to the previous node in the chain.
The DAG is a directed graph, meaning that the edges connecting the nodes
have a direction.
* The DAG is acyclic, meaning that there are no cycles in the graph.
* The DAG is useful for efficiently storing and querying the causality between
events in a distributed system. It allows for fast insertion and deletion of
nodes, and efficient querying of the relationship between nodes.

Functions for inserting, deleting, and querying nodes in the DAG are
provided to facilitate the implementation of the ssb-clock feature. These
functions are described in more detail in the following sections.

````
class DAG {
    private table: HashTable<Hash, Node>

    constructor() {
        this.table = new HashTable<Hash, Node>()
    }

    addNode(id: Hash, payload: Payload, parents: Hash[], ROF: RandomOracleFunction): void {
        let bcs = new BloomClockStamp()
        for (const parent of parents) {
          bcs = join(bcs, this.table.get(parent).stamp)
        }
        bcs = increment(bcs)
        const node = new Node(id, payload, parents, bcs)
        
        this.table.add(id, node)
        
    }

    getNode(id: string): Node | null {
        return this.table.get(id)
    }

    removeNode(id: string): void {
        this.table.remove(id)
    }
}

class Node {
    hash: string
    payload: Payload
    parents: string[]
    stamp: BloomClockStamp

    constructor(hash: Hash, payload: Payload, parents: string[], stamp: BloomClockStamp) {
        this.hash = hash
        this.payload = payload
        this.parents = parents
        this.stamp = stamp
    }
}

````
### The Merkle Tree

````typescript

function createMerkleTree(dagRoot: Node): MerkleTreeNode {
  let leafs = []
  let nextNode = dagRoot
  while (nextNode.hasChildren()) {
    const children = nextNode.getChildren()
    if (children.length > 1) {
      let branchRoots = []
      
      children.sort()
      
      for (const child of children) {
        branchRoots.push(createMerkleTree(child))
      }
      
      leafs.push(createMerkleTreeFromLeaves(branchRoots))
    } else {
      let nextNode = children[0]
      leafs.push(new MerkleTreeNode(nextNode))
    }
  }

  return createMerkleTreeFromLeaves(branchHashes)
}

````

## Protocol-specific considerations

For each protocol, a description of any dependencies on other protocols or
components, such as the SSB core protocol or other synchronization protocols.

* A discussion of the data structures and algorithms used in the
  implementation of each protocol, such as the SELECT function for determining
  which peers are allowed to increment the counter.
* Considerations for performance and scalability, such as the impact on
  network traffic and the ability to handle large numbers of concurrent updates.
* A discussion of any security considerations, such as the potential for
  malicious actors to manipulate the global counter or attempt to cheat the
  proof of delay.

## Integration with SSB

* A description of how the protocols fit into the overall architecture of SSB,
  including any modifications or extensions to the core protocol that may be
  required.
* A discussion of any additional security measures that may be necessary when
  integrating the protocols with SSB, such as the use of cryptographic
  signatures or additional validation checks.

## Deployment and maintenance

* A description of the process for deploying the protocols in a production
  environment, including any infrastructure or resources that will be required.
* A discussion of any ongoing maintenance that will be needed, such as
  regular updates or the handling of bugs or issues that may arise.
* A consideration of any potential challenges or issues that may arise during
  deployment or maintenance, such as the need to migrate data or coordinate
  updates across multiple peers.
