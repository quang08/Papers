# [Cassandra - A Decentralized Structured Storage System](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf)
	
# 1. Data Model:
- Cassandra’s data model is a distributed, multi-dimensional map indexed by a key, where each row key is a string. Operations under a single row key are atomic per replica.
- Column Families group columns in a way similar to Bigtable’s system. Cassandra offers Simple and Super Column Families, where the latter represents a hierarchy of columns.
- Columns within a column family can be sorted either by name or timestamp, which is beneficial for applications like Inbox Search where time-ordering is important.
- Eg:
  
Row Key (String) | Column Family | Column Name | Column Value
-----------------|---------------|-------------|------------------------------
"user:1001"      | "profile"     | "name"      | "Alice Smith"
"user:1001"      | "profile"     | "email"     | "alice@example.com"
"user:1001"      | "profile"     | "age"       | 28
"user:1001"      | "posts"       | "post:1"    | "Hello, Cassandra!"
"user:1001"      | "posts"       | "post:2"    | "Learning about NoSQL databases."
"user:1002"      | "profile"     | "name"      | "Bob Johnson"
"user:1002"      | "profile"     | "email"     | "bob@example.com"
"user:1002"      | "profile"     | "age"       | 35

| Explanation |
|---------------|
| The row keys are strings like "user:1001" and "user:1002"  |
| Each row can have multiple column families (like "profile" and "posts"). |
| Within each column family, there can be multiple columns with names and values |

# 2. System Architecture:
- Cassandra uses techniques like partitioning (splitting data), replication (copying data), and failure handling (managing node failures) to manage read and write requests. When writing, it waits for a certain number of copies (replicas) to confirm success; when reading, it ensures consistency by contacting replicas.
  
**=> Semi Synchronous Data Replication for Writes: waits for a majority of replicas to confirm whether the data  has been written. After that, it sends an acknowledgement to the Client confirming all writes has been written. If not all replicas in the system have written the data yet, Cassandra continues to replicate the data asynchronously.**
**=> For reads, uses quorum mechanism based on the consistency level required by the client. It may read from the closes replica or contact multiple replicas and wait for quorum of responses.**

## 2.1 Partitioning:

- Cassandra uses consistent hashing to distribute data across nodes, where data is assigned based on hash values. The consistent hashing approach allows for incremental scalability.
- Challenges include non-uniform data distribution and load distribution, which are addressed by either assigning nodes to multiple positions on the hash ring or through dynamic load adjustments.
  
### Example of Consistent Hashing
- Imagine a circular ring representing all possible hash values. This is how Cassandra organizes its data distribution:
  - **Nodes**: These are the servers in the Cassandra cluster. In our diagram, we have Nodes A, B, C, and D represented by green circles on the ring.
  - **Data items**: These are the pieces of data you want to store. In our example, we have Data Item X, represented by the orange dot.
The cycle (consistent hashing):
```
    1. When you want to store a data item, Cassandra hashes its key.
    2. This hash value determines the item's position on the ring.
    3. Cassandra then moves clockwise from this position to find the first node.
    4. This node becomes responsible for storing the data item.
```

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 400" width="400">
  <circle cx="200" cy="200" r="180" fill="none" stroke="#333" stroke-width="2"/>
  <circle cx="200" cy="20" r="10" fill="#4CAF50"/>
  <circle cx="350" cy="200" r="10" fill="#4CAF50"/>
  <circle cx="200" cy="380" r="10" fill="#4CAF50"/>
  <circle cx="50" cy="200" r="10" fill="#4CAF50"/>
  <text x="190" y="10" font-family="Arial" font-size="12">Node A</text>
  <text x="360" y="205" font-family="Arial" font-size="12">Node B</text>
  <text x="190" y="395" font-family="Arial" font-size="12">Node C</text>
  <text x="10" y="205" font-family="Arial" font-size="12">Node D</text>
  <circle cx="275" cy="75" r="5" fill="#FF5722"/>
  <text x="285" y="80" font-family="Arial" font-size="12">Data Item X</text>
  <path d="M200,200 L275,75" stroke="#FF5722" stroke-width="2" stroke-dasharray="5,5"/>
  <path d="M275,75 A180,180 0 0,1 350,200" stroke="#FF5722" stroke-width="2" stroke-dasharray="5,5"/>
</svg>

- In our diagram, Data Item X is hashed to a position between Node A and Node B. Moving clockwise, we find that Node B is the first node we encounter, so Node B becomes responsible for storing Data Item X. This system allows for easy scaling:
```
- If we add a new node, it only affects the data stored in its immediate neighbor.
- If we remove a node, only its data needs to be redistributed to its neighbor.
```

### Non-uniform data distribution and load distribution:
- Non-uniform data distribution occurs when data isn't evenly spread across all nodes. Load distribution refers to the amount of work each node has to do.
- **Example of non-uniform distribution:**
Let's say we have four nodes: A, B, C, and D. In an ideal scenario, each would handle 25% of the data. However, due to the random nature of hashing, we might end up with:
```
    Node A: 40% of data
    Node B: 10% of data
    Node C: 30% of data
    Node D: 20% of data
```
- To address this, Cassandra can:
  - Assign nodes to multiple positions on the ring **(virtual nodes)**, which helps spread the load more evenly.
  - Dynamically adjust node positions based on their current load, moving lightly loaded nodes to take on more work from heavily loaded nodes.
  
### Virtual Nodes
- We'll consider a simplified Cassandra cluster with just two physical nodes, but using virtual nodes to distribute the data more evenly.
- **Physical Nodes:**
```
Node A (represented by green circles)
Node B (represented by blue circles)
```
- **Virtual Nodes:**
  - Instead of each physical node occupying just one position on the ring, we assign each node to multiple positions:
```
Node A has virtual nodes A1, A2, and A3
Node B has virtual nodes B1, B2, and B3
```

- **Data Distribution:**
  - Let's consider three data items: **X, Y, and Z** (represented by orange dots).
Without virtual nodes:
  - If Node A was only at position A1 and Node B at B1, all three data items might end up on Node A, causing uneven distribution.

  - **With virtual nodes:**
```
Data item X is closest to virtual node A2, so it's stored on physical Node A.
Data item Y is closest to virtual node A1, so it's also stored on physical Node A.
Data item Z is closest to virtual node B2, so it's stored on physical Node B.
```

- **Benefits:**
  - Even though we only have two physical nodes, the use of virtual nodes has allowed for a more even distribution of data.
  - Node A is responsible for 2 data items, and Node B for 1, which is more balanced than all 3 potentially ending up on one node.
  - If we added or removed a physical node, the rebalancing would affect smaller portions of data, making the process smoother and less impactful on performance.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 400" width="400">
  <circle cx="200" cy="200" r="180" fill="none" stroke="#333" stroke-width="2"/>
  
  <!-- Node A -->
  <circle cx="200" cy="20" r="10" fill="#4CAF50"/>
  <circle cx="320" cy="120" r="10" fill="#4CAF50"/>
  <circle cx="80" cy="120" r="10" fill="#4CAF50"/>
  <text x="190" y="10" font-family="Arial" font-size="12">A1</text>
  <text x="330" y="125" font-family="Arial" font-size="12">A2</text>
  <text x="60" y="125" font-family="Arial" font-size="12">A3</text>
  
  <!-- Node B -->
  <circle cx="350" cy="200" r="10" fill="#2196F3"/>
  <circle cx="200" cy="350" r="10" fill="#2196F3"/>
  <circle cx="120" cy="280" r="10" fill="#2196F3"/>
  <text x="360" y="205" font-family="Arial" font-size="12">B1</text>
  <text x="190" y="365" font-family="Arial" font-size="12">B2</text>
  <text x="100" y="295" font-family="Arial" font-size="12">B3</text>
  
  <!-- Data items -->
  <circle cx="275" cy="75" r="5" fill="#FF5722"/>
  <circle cx="150" cy="50" r="5" fill="#FF5722"/>
  <circle cx="300" cy="300" r="5" fill="#FF5722"/>
  <text x="285" y="80" font-family="Arial" font-size="12">X</text>
  <text x="160" y="55" font-family="Arial" font-size="12">Y</text>
  <text x="310" y="305" font-family="Arial" font-size="12">Z</text>
</svg>

## 2.2 Replication:

### Replication for high availability:
- Each data item is replicated at *N* hosts, where *N* is the configurable replication factor.
- A coordinator node is in charge of replication for data items within its range.

### Replication strategies:
- **"Rack Unaware":** Non-coordinator replicas are chosen randomly.
  - Cassandra randonly selects replicas without considering the physical location of nodes.
  - Example: If there are three replicas, Cassandra might put one copy in New York, one in Paris and one in Tokyo, without worrying about data center or rack placement
- **"Rack Aware":** Considers data center topology.
  - This strategy considers the *physical layout of servers* *within a data center*. It tries to place replicas on different racks to reduce the risk of losing all copies due to a rack failure.
  - Example: If a data center has multiple racks of servers, Cassandra might place one copy of the data in Rack 1, another in Rack 3, and the last one in Rack 5 to ensure data is spread out across racks.
- **"Datacenter Aware":** More complex strategy considering multiple data centers.
  - Different from Rack Awarem, this strategy considers the layout across multiple data centers. It ensures that replicas are placed in different data centers to prevent a full data center failure from causing data loss.

### Leadership and range management:
- Cassandra uses **Zookeeper** to elect a leader among nodes.
  - It helps keep track of which node in a cluster is performing specific roles
  - With that, it helps the cluster decide which node should act as the leader when needed
- The leader ensures no node is responsible for more than *N-1* ranges in the ring.
  - Cassandra divides its data into "ranges" of data. The *leader nodes* ensures that no single node is overloaded by being responsible for too many ranges
  - Example: if *N* (the replication factor) is 3, the leader ensures that each node only handles at most 2 ranges, spreading the workload evenly across the cluster
- Nodes maintain metadata about their responsible ranges locally.

### Fault tolerance:
- Zookeeper helps maintain system state even if nodes crash and come back up.
- Cassandra can handle node failures and network partitions by *relaxing quorum requirements.*
  - In case of node failures or network partitions, Cassandra can temporarily lower the required number of nodes to complete a request. This allows the system to remain operational even if some nodes are unavailable.

### Multi-data center support:
- Data is replicated across multiple data centers.
- Storage nodes are spread across multiple data centers.
- Data centers are connected through high-speed network links.

### Preference list:
- For each key, a preference list is constructed.
  - This is an ordered list of nodes where replicas for a given key are stored, which determines which data centers and nodes will store replicas.
  - Example: If Cassandra decides to store a key “User123” at three different nodes, the preference list might look like:
```
	1.	Node A (in New York data center)
	2.	Node B (in San Francisco data center)
	3.	Node C (in London data center)
```

### Failure handling:
- Cassandra is designed to handle data center failures without outages.
- It can cope with power outages, cooling failures, network failures, and natural disasters.

### Client options:
- Clients have various options for how data needs to be replicated.

### System awareness:
- *Every node is aware of every other node in the system.*
- Nodes know the ranges they are responsible for.

**=> This design allows Cassandra to provide high availability, durability, and fault tolerance across multiple data centers while maintaining flexibility in replication strategies.**


## 2.3 Membership:
- Membership in Cassandra is managed using a *Gossip-based protocol (Scuttlebutt)*, which efficiently spreads membership information and control state across nodes. *"very efficient anti-entropy Gossip based mechanism."*
  - **Gossip Protocol:** Gossip is a decentralized communication protocol where nodes periodically exchange state information with each other. In a gossip protocol, each node randomly selects another node to share information with, allowing information to propagate throughout the network.
  - **Anti-entropy:** This term suggests that the protocol works to *reduce disorder* or *inconsistency* in the system. In distributed systems, anti-entropy mechanisms help ensure that all nodes eventually converge to a consistent state.
- Failure detection uses an Accrual Failure Detector, which assigns a suspicion level *(Φ value)* to determine whether a node is up or down, rather than a simple binary status.
- In Cassandra, the Gossip protocol is not limited to just maintaining membership information. It's also used to disseminate other system-related control state. This means that nodes can share various types of system information efficiently using the same mechanism.
  - But the primary purpose of this protocol in Cassandra is to keep track of which nodes are part of the cluster. This is crucial for distributed operations, data replication, and maintaining the overall health of the cluster.

**=> This approach allows Cassandra to scale effectively, handle node joins and leaves gracefully, and maintain overall system consistency with minimal overhead.**

## 2.4 Bootstrapping: Initializing and Integrating a New Node into An Existing Cluster
### Node Initialization
- When a node starts for the first time, it chooses a random token for its position in the ring.
- This mapping is persisted locally on disk and in Zookeeper for fault tolerance.
- The token information is gossiped around the cluster, allowing all nodes to know about each other's positions.

### Joining A Cluster
- A new node reads its configuration file containing a list of "seed" nodes (initial contact points) within the cluster.
- These seeds provide the new node with information about the cluster's current state.

### Fault Tolerance
- The system is designed to handle various types of node outages, including disk failures and CPU issues.
- Node outages are typically temporary and don't require permanent changes to the cluster structure.
- Every message contains the cluster name of the Cassandra instance.
**-> This prevents accidental connections between different Cassandra clusters**

### Explicit Node Management:
- An explicit mechanism is used to add or remove nodes from a Cassandra instance. This is done through an administrative command-line tool or a browser interface.


## 2.5 Scaling the Cluster:

- As the cluster scales, new nodes are assigned tokens to take over portions of the data previously held by other nodes.
- Cassandra uses efficient data streaming methods, such as kernel-to-kernel copying, to transfer data during bootstrapping and scaling. The system has been shown to handle data transfer at rates of around 40 MB/s.

### Practical Experiences: Facebook Inbox Search

- Inbox Search uses Cassandra to store and index large volumes of user messages, facilitating two types of searches: term-based search and interaction-based search.
- The system maintains per-user indices with search data stored in Super Column Families, organized either by words in messages or by recipients.
- Performance: The system currently handles more than 50TB of data across 150 nodes, providing read latencies as low as 7.69ms (min) and 18.27ms (median) for term searches, showcasing its efficiency and scalability.
