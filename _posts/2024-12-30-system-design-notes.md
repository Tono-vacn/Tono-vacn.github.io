---
layout: post
title: System-Design-Notes
subtitle: Notes for system design
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/system_design_pic.jpg
# share-img: /assets/img/path.jpg
tags: [
    System Deign,
]
author: Yuchen Jiang
readtime: true
---

A simple note for system design.

## From *System Design Interview* by Alex Xu

### K-V Store

#### Design scopes

- The size of a key-value pair is small: less than 10 KB.
- Ability to store big data.
- High availability: The system responds quickly, even during failures.
- High scalability: The system can be scaled to support large data set.
- Automatic scaling: The addition/deletion of servers should be automatic based on traffic.
- Tunable consistency.
- Low latency.

#### CAP Theorem

Check the detailed explanation if not familiar with CAP theorem.

- CP (consistency and partition tolerance) systems: a CP key-value store supports consistency and partition tolerance while sacrificing availability.

- AP (availability and partition tolerance) systems: an AP key-value store supports availability and partition tolerance while sacrificing consistency.

- CA (consistency and availability) systems: a CA key-value store supports consistency and availability while sacrificing partition tolerance. Since network failure is unavoidable, a distributed system must tolerate network partition. Thus, a **CA system cannot exist in real-world applications**.

#### System Components

- Data partition: split the data into smaller partitions and store them in
multiple servers
  - Distribute data across multiple servers evenly
  - Minimize data movement when nodes are added or removed
  - Using **consistent hashing** to distribute data
    - Automatic scaling: servers could be added and removed automatically depending on the load.
    - Heterogeneity: the number of virtual nodes for a server is proportional to the server capacity. For example, servers with higher capacity are assigned with more virtual nodes.
- Data replication: data must be replicated asynchronously over N servers, where N is a configurable parameter
  - after a key is mapped to a position on the hash ring, walk clockwise from that position and choose the first N servers on the ring to store data copies.
  - With virtual nodes, the first N nodes on the ring may be owned by fewer than N physical servers. To avoid this issue, we only choose **unique servers** while performing the clockwise walk logic.
- Consistency: Using Quorum consensus to guarantee consistency
  - Definitions:
    - R: read quorum of size R
    - W: write quorum of size W
    - N: the total number of nodes
    - If R = 1 and W = N, the system is optimized for a fast read.
    - If W = 1 and R = N, the system is optimized for fast write.
    - If W + R > N, strong consistency is guaranteed (Usually N = 3, W = R = 2).
  - consistency model:
    - Strong consistency
    - Weak consistency: subsequent read operations may not see the most updated value
    - **Eventual consistency**: this is a specific form of weak consistency. Given enough time, all updates are propagated, and all replicas are consistent. Adopted by Dynamo and Cassandra
- Inconsistency resolution: Versioning and vector clocks
  - Versioning: treating each data modification as a new immutable version of data
  - Vector clocks: a [server, version] pair associated with a data item. It can be used to check
    - represented by D([S1, v1], [S2, v2], …, [Sn, vn]), where D is a data item, v1 is a version counter, and s1 is a server number, etc.
    - If data item D is written to server Si
      - Increment vi if [Si, vi] exists
      - Otherwise, create a new entry [Si, 1]
    - set a threshold for the length, if it exceeds the limit, the oldest pairs are removed
- Handling failures
  - Failure detection: gossip protocol
    - Each node maintains a node membership list, which contains member IDs and heartbeat counters.
    - Each node periodically increments its heartbeat counter
    - Each node periodically sends heartbeats to a set of random nodes, which in turn propagate to another set of nodes
    - Once nodes receive heartbeats, membership list is updated to the latest info
    - If the heartbeat has not increased for more than predefined periods, the member is considered as offline
  - temporary failure handling
    - replica set: a set of nodes that store the same data. Any strict quorum standards should be performed on the replica set.
    - sloppy quorum: Instead of enforcing the quorum requirement, the system chooses the first W healthy servers for writes and first R healthy servers for reads on the hash ring. Offline servers are ignored.
    - hinted handoff: If a server is unavailable due to network or server failures, another server will process requests temporarily. When the down server is up, changes will be pushed back to achieve data consistency.

- System architecture diagram
- Write path
- Read path