---
layout: post
title: Notes for Data-Intensive Applications
subtitle: Notes for data-intensive applications
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/system_design_pic.jpg
# share-img: /assets/img/path.jpg
tags: [
    System Deign,
    Distributed System,
    Data-Intensive Applications,
]
author: Yuchen Jiang
readtime: true
---

A simple note for data-intensive applications.

From *Designing Data-Intensive Applications* by Martin Kleppmann

## Data Replication

*single-leader*, *multi-leader*, and *leaderless* replication

### leader-based replication: 
  
  - main steps:
    1. One of the replicas is designated the leader (also known as master or primary).
    When clients want to write to the database, they must send their requests to the
    leader, which first writes the new data to its local storage.
    
    2. The other replicas are known as followers (read replicas, slaves, secondaries, or hot
    standbys). Whenever the leader writes new data to its local storage, it also sends
    the data change to all of its followers as part of a replication log or change stream.
    Each follower takes the log from the leader and updates its local copy of the data base accordingly, by applying all writes in the same order as they were processed
    on the leader.
    
    3. When a client wants to read from the database, it can query either the leader or any of the followers. However, writes are only accepted on the leader (the follow‐
    ers are read-only from the client’s point of view).

  - Synchronous Versus Asynchronous Replication
    - Synchronous: the leader waits until follower 1 has confirmed that it received the write before reporting success to the user, and before making the write visible to other clients
    - Asynchronous: the leader sends the message, but doesn’t wait for a response from the follower
    - semi-synchronous: one of the followers is synchronous, and the others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous.

  - Setting Up New Followers (Consistency and Durability):
    1. Take a consistent snapshot of the leader’s database at some point in time—if possible, without taking a lock on the entire database. 
    
    2. Copy the snapshot to the new follower node.
    
    3. The follower connects to the leader and requests all the data changes that have
    happened since the snapshot was taken. This requires that the snapshot is associated with an exact position in the leader’s replication log. 
    
    4. When the follower has processed the backlog of data changes since the snapshot,
    we say it has *caught up*. It can now continue to process data changes from the
    leader as they happen.

  - Handling Node Outages (availability):
    - Follower failure: Catch-up recovery
      - from its log, it knows the last transaction that was processed before the fault occurred. Thus, the follower can connect to the leader and request all the data changes that occurred during the time when the follower was disconnected.
    - Leader failure: Failover
      - failover: the process of choosing a new leader when the old leader is no longer available.
      - automatic failover process:
        1. Determining that the leader has failed: heartbeat and timeout
        2. Choosing a new leader: election process, quorum-based consensus algorithm
        3. Reconfiguring the system to use the new leader: The system needs to ensure that the old leader becomes a follower and recognizes the new leader

#### leader-based replication implementation

- Statement-based replication
  - cons:
    - Any statement that calls a nondeterministic function, such as NOW() to get the current date and time or RAND() to get a random number, is likely to generate a different value on each replica.
    - If statements use an autoincrementing column, or if they depend on the existing data in the database, must be executed in exactly the same order on each replica, or else they may have a different effect.
    -  Statements that have side effects (e.g., triggers, stored procedures, user-defined functions) may result in different side effects occurring on each replica, unless the side effects are absolutely deterministic.
  - not that commonly used now in practice
- Write-ahead log (WAL) shipping
  - used in PostgreSQL and Oracle
  - use the exact same log to build a replica on another node: besides writing the log to disk, the leader also sends it across the network to its followers.
  - cons:
    - log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk blocks. 
    - replication closely coupled to the storage engine
    - typically not possible to run different versions of the database software on the leader and the followers.
    - If the replication protocol allows the follower to use a newer software version than the leader, you can perform a **zero-downtime upgrade** of the database software by first upgrading the followers and then **performing a failover to make one of the upgraded nodes the new leader**. If the replication protocol does not allow this version mismatch, as is often the case with WAL shipping, such upgrades **require downtime**.
- Logical (row-based) log replication
  - use different log formats for replication and for the storage engine, which allows the replication log to be decoupled from the storage engine internals.
  - A logical log for a relational database is usually a sequence of records describing writes to database tables at the granularity of a row
    - For an inserted row, the log contains the new values of all columns
    - For a deleted row, the log contains enough information to uniquely identify the row that was deleted (usually the primary key)
    - For an updated row, the log contains enough information to uniquely identify the updated row, and the new values of all columns (or at least the new values of all columns that changed).
    - For transaction: A transaction that modifies several rows generates several such log records, followed by a record indicating that the transaction was committed.
- Trigger-based replication
  - move replication up to the application layer
  - A trigger lets you register custom application code that is automatically executed when a data change (write transaction) occurs in a database system.
  - The trigger has the opportunity to log this change into a separate table, from which it can be read by an external process. That external process can then apply any necessary application logic and replicate the data change to another system.

#### Problems with *Replication Lag*

Appearently syncrounous replication is impratical for Leader-based replication aiming to handle workloads that consist of mostly reads and only a small percentage of writes (a common pattern on the web).

If an application reads from an *asynchronous* follower, it may see outdated information if the follower has fallen behind. This leads to apparent inconsistencies in the database: if you run the same query on the leader and a follower at the same time, you may get different results, because not all writes have been reflected in the follower.

Eventual consistency: if you stop writing to the
database and wait a while, the followers will eventually catch up and become consistent with the leader.

Three possible problems that are likely to occur when there is replication lag

- Reading Your Own Writes: A user makes a write, followed by a read from a stale replica. To prevent this anomaly, we need read-after-write consistency.
  - read-after-write consistency / read-your-writes consistency: guarantee that if the user reloads the page, they will always see any updates they submitted themselves.
  - It makes no promises about other users: other users’ updates may not be visible until some later time.
  - possible implementations:
    - When reading something that the user may have modified, read it from the leader; otherwise, read it from a follower. (For example, user profile information on a social network is normally only editable by the owner of the profile, not by anybody else. Thus, a simple rule is: always read the user’s own profile from the leader, and any other users’ profiles from a follower.)
    - Use other criteria to decide whether to read from the leader. (track the time of the last update, for one minute after the last update, make all reads from the leader)
    - The client can remember the timestamp of its most recent write—then the system can ensure that the replica serving any reads for that user reflects updates at least until that timestamp. If a replica is not sufficiently up to date, either the read can be handled by another replica or the query can wait until the replica has caught up.- Monotonic Reads
  - lesser guarantee than strong consistency, but a stronger guarantee than eventual consistency.
  - monotonic reads only means that if one user makes several reads in sequence, they will not see time go backward—i.e., they will not read older data after having previously read newer data.
  - possible implementation:
    - each user always makes their reads from the same replica (different users can read from different replicas).
- Consistent Prefix Reads
  - This guarantee says that if a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order.
  - in many distributed databases, different partitions operate independently, so there is no global ordering of writes: when a user reads from the database, they may see some parts of the database in an older state and some in a newer state.
  - possible solution: make sure that any writes that are causally related to each other are written to the same partition

### Multi-Leader Replication

Leader-based replication downside: one leader, and all writes must go through it.

multi-leader configuration / master–master or active/active replication:  allow more than one node to accept writes. Replication still happens in the same way: each node that processes a write must forward that data change to all the other nodes.

#### Use Cases for Multi-Leader Replication

- Multi-datacenter operation
  - ![Multi-datacenter](/assets/img/Multi-Leader.png)
  - have a leader in each datacenter, Within each datacenter, regular leader–follower replication is used; between datacenters, each datacenter’s leader replicates its changes to the leaders in other datacenters.
  - the same data may be concurrently modified in two different datacenters, and those write conflicts must be resolved.
- Clients with offline operation
  - if you have an application that needs to continue to work while it is disconnected from the internet, like calendar. If you make any changes while you are offline, they need to be synced with a server and your other devices when the device is next online.
  - every device has a local database that acts as a leader (it accepts write requests), and there is an asynchronous multi-leader replication process (sync) between the replicas of your calendar on all of your devices.
  - this setup is essentially the same as multi-leader replication between datacenters, taken to the extreme: each device is a “datacenter,” and the network connection between them is extremely unreliable.
- Collaborative editing
  - like Google Docs, Etherpad
  - When one user edits a document, the changes are instantly applied to their local replica (the state of the document in their web browser or client application) and asynchronously replicated to the server and any other users who are editing the same document.

#### Handling Write Conflicts

##### Conflict avoidance
  - if the application can ensure that all writes for a particular record go through the same leader, then conflicts cannot occur.
##### Converging toward a consistent state
  - every replication scheme must ensure that the data is eventually the same in all replicas.
  - possible solutions:
    - Give each write a unique ID (e.g., a timestamp, a long random number, a UUID, or a hash of the key and value), pick the write with the highest ID as the winner, and throw away the other writes.
      - If a timestamp is used, this technique is known as last write wins (LWW).
      - dangerously prone to data loss
    - Give each replica a unique ID, and let writes that originated at a higher-numbered replica always take precedence over writes that originated at a lower-numbered replica.
      -  also implies data loss
    - Somehow merge the values together
    - Record the conflict in an explicit data structure that preserves all information, and write application code that resolves the conflict at some later time (perhaps by prompting the user).
##### Custom conflict resolution logic

most appropriate way of resolving a conflict may depend on the application

two choices:
- On write: 
  - As soon as the database system detects a conflict in the log of replicated changes, it calls the conflict handler.
- On read: 
  - When a conflict is detected, all the conflicting writes are stored.
  - The next time the data is read, The application may prompt the user or automatically resolve the conflict, and write the result back to the database.

Also, conflict resolution usually applies at the level of an individual row or document, not for an entire transaction

if you have a transaction that atomically makes several different writes (see Chapter 7), each write is still considered separately for the purposes of conflict resolution.

##### Automatic Conflict Resolution

- Conflict-free replicated datatypes (CRDTs)
- Mergeable persistent data structures
- Operational transformation: Google Docs

#### Multi-Leader Replication Topologies

circular, star topology, all-to-all / full mesh

- problem for circular and star topology:
  - if just one node fails, it can interrupt the flow of replication messages between other nodes
- problem for all-to-all / full mesh:
  - some network links may be faster than others (e.g., due to network congestion), with the result that some replication messages may “overtake” others

### Leaderless Replication

#### Writing to the Database When a Node Is Down

To prevent read stale data from a node which is previously down, when a client reads from the database, it doesn’t just send its request to one replica: read requests are also sent to several nodes in parallel.

Version numbers are used to determine which value is newer

##### Read repair and anti-entropy

After an unavailable node comes back online, how does it catch up on the writes that it missed?

- Read repair
  - When a client makes a read from several nodes in parallel, it can detect any stale responses.
  - The client can then send the latest value back to the node that had the stale value, and that node can update its local copy.
- Anti-entropy
  - a background process that constantly looks for differences in the data between replicas and copies any missing data from one replica to another
  - does not copy writes in any particular order
  - may be a significant delay before data is copied

##### Quorums for reading and writing

quorum reads and writes:
- w + r > n
- A common choice is to make n an odd number (typically 3 or 5) and to set w = r = (n + 1) / 2 (rounded up).

#### Limitations of Quorum Consistency

- If a *sloppy quorum* is used, the w writes may end up on different nodes than the r reads, so there is no longer a guaranteed overlap between the r nodes and the w nodes
- If two writes occur concurrently, it is not clear which one happened first. the only safe solution is to merge the concurrent writes
- If a write happens concurrently with a read, the write may be reflected on only some of the replicas. In this case, it’s undetermined whether the read returns the old or the new value.
- If a write succeeded on some replicas but failed on others (for example because the disks on some nodes are full), and overall succeeded on fewer than w replicas, it is not rolled back on the replicas where it succeeded.
- If a node carrying a new value fails, and its data is restored from a replica carry‐ ing an old value, the number of replicas storing the new value may fall below w, breaking the quorum condition.

#### Sloppy Quorums and Hinted Handoff

sloppy quorum:
  - writes and reads still require w and r successful responses, but those may include nodes that are not among the designated n “home” nodes for a value.
  - network interruption can easily cut off a client from a large number of database node, under this circumstance, accept writes anyway, and write them to some nodes that are reachable but aren’t among the n nodes on which the value usually lives


hinted handoff:
  - Once the network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to the appropriate “home” nodes.


Sloppy quorums are particularly useful for increasing write availability: 
- as long as any w nodes are available, the database can accept writes. 
- when w + r > n, you cannot be sure to read the latest value for a key, because the latest value may have been temporarily written to some nodes outside of n

#### Detecting Concurrent Writes

The problem is that events may arrive in a different order at different nodes, due to variable network delays and partial failures.

If each node simply overwrote the value for a key whenever it received a write request from a client, the nodes would become permanently inconsistent, the replicas should converge toward the same value. 

- Last Write Wins (discarding concurrent writes)
  - Use a timestamp to determine which write is the latest: last write wins (LWW)
    - only one of the writes will survive and the others will be silently discarded
    - may even drop writes that are not concurrent
  - only safe way of using a database with LWW is to ensure that a key is only written once and thereafter treated as immutable
- The “happens-before” relationship and concurrency
  - An operation A happens before another operation B if B knows about A, or depends on A, or builds upon A in some way.
  - Three possibilities:
    - A happens before B
    - B happens before A
    - A and B are concurrent
  - Capturing the happens-before relationship:
    1. The server maintains a version number for every key, increments the version number every time that key is written, and stores the new version number along with the value written.
    2. When a client reads a key, the server returns all values that have not been overwritten, as well as the latest version number. A client must read a key before writing.
    3. When a client writes a key, it must include the version number from the prior read, and it must merge together all values that it received in the prior read. (The response from a write request can be like a read, returning all current values, which allows us to chain several writes like in the shopping cart example.)
    4. When the server receives a write with a particular version number, it can overwrite all values with that version number or below (since it knows that they have been merged into the new value), but it must keep all values with a higher version number (because those values are concurrent with the incoming write).
  - Merging concurrently written values
    - clients have to clean up afterward by merging the concurrently written values.
    - a reasonable approach to merging siblings is to just take the union.
    - tombstone: an item cannot simply be deleted from the database when it is removed; instead, the system must leave a marker with an appropriate version number to indicate that the item has been removed when merging siblings.
- Version vectors
  - we need to use a version number per replica as well as per key.
  - Each replica increments its own version number when processing a write, and also keeps track of the version numbers it has seen from each of the other replicas. 
  - The collection of version numbers from all the replicas is called a version vector.
  - The version vector structure ensures that it is safe to read from one replica and subsequently write back to another replica.
















