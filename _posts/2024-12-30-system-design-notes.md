---
layout: post
title: System Design Notes
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
  - permanent failure handling
    - anti-entropy protocol: Merkle tree
    - check the detailed structure of Merkle tree if needed
    - Merkle tree is used for inconsistency detection and minimizing the amount of data transferred

- System architecture diagram
  ![sys_diagram](/assets/img/sys_diagram.png)
  - Clients communicate with the key-value store through simple APIs: get(key) and put(key, value).
  - A coordinator is a node that acts as a proxy between the client and the key-value store.
  - Nodes are distributed on a ring using consistent hashing.
  - The system is completely decentralized so adding and moving nodes can be automatic.
  - Data is replicated at multiple nodes.
  - There is no single point of failure as every node has the same set of responsibilities.

  ![node](/assets/img/node_diagram.png)
- Write path
  ![write_path](/assets/img/write_path.png)
  - write request is persisted on a commit log file
  - Data is saved in the memory cache
  - When the memory cache is full or reaches a predefined threshold, data is flushed to SSTable on disk
- Read path
  ![read_path](/assets/img/read_path.png)


### Unique ID Generator

#### Requirements

- IDs must be unique.
- IDs are numerical values only.
- IDs fit into 64-bit.
- IDs are ordered by date.
- Ability to generate over 10,000 unique IDs per second.

#### Probable solutions

##### Multi-master replication

![master](/assets/img/master.png)

- uses the databases’ auto_increment feature
- increase it by k, where k is the number of database servers in use

- Hard to scale with multiple data centers
- IDs do not go up with time across multiple servers.
- It does not scale well when a server is added or removed.

##### UUID

- let each web server contain an UUID generator

- Pros:
  - Generating UUID is simple. No coordination between servers is needed so there will not be any synchronization issues.
  - The system is easy to scale because each web server is responsible for generating IDs they consume. ID generator can easily scale with web servers.
- Cons:
  - IDs are 128 bits long, but our requirement is 64 bits.
  - IDs do not go up with time.
  - IDs could be non-numeric.

##### Ticket server

![ticket_server](/assets/img/ticket_server.png)
- Single point of failure.

##### Snowflake

![snowflake](/assets/img/snowflake.png)

- Sign bit: 1 bit. It will always be 0. This is reserved for future uses. It can potentially be used to distinguish between signed and unsigned numbers.
- Timestamp: 41 bits. Milliseconds since the epoch or custom epoch. We use Twitter snowflake default epoch 1288834974657, equivalent to Nov 04, 2010, 01:42:54 UTC.
- Datacenter ID: 5 bits, which gives us 2 ^ 5 = 32 datacenters.
- Machine ID: 5 bits, which gives us 2 ^ 5 = 32 machines per datacenter.
- Sequence number: 12 bits. For every ID generated on that machine/process, the sequence number is incremented by 1. The number is reset to 0 every millisecond.

#### Furhter problems

- Clock synchronization: Network Time Protocol (NTP)
- Section length tuning
- High availability


### URL Shortener

#### API Design

- URL shortening: POST
- URL redirection: GET

##### Redirecting

changes the short URL to the long URL with 301 redirect

![shorten](/assets/img/shorten.png)

- 301 vs 302
  - 301: permanent redirect, subsequent requests for the same URL will not be sent to the URL shortening service.
  - 302: temporary redirect, subsequent requests for the same URL will be sent to the URL shortening service first.
  - For server load considerations, 301 is preferred.
  - For analytics, 302 is preferred.

##### Shortening

Using Hashing
  - Each longURL must be hashed to one hashValue.
  - Each hashValue can be mapped back to the longURL.

Using Database
  - memory resources are limited and expensive
  - relational database

Handle collisions
  - most existing Hash functions produce too-long hash values
  - shorten hash results might have collisions
  - How to fix:
    ![solution](/assets/img/hash_solution.png)
  - frequently check the database for collisions can be expensive
    - use bloom filters

Use Base62 conversion

- check the detailed explanation if not familiar with Base62 conversion

![comparing](/assets/img/comp.png)
  
#### Possible process

- Assuming the input longURL is: https://en.wikipedia.org/wiki/Systems_design
- Unique ID generator returns ID: 2009215674938.
- Convert the ID to shortURL using the base 62 conversion. ID (2009215674938) is converted to “zn9edcu”.
- Save ID, shortURL, and longURL to the database


### Web Crawler

#### Possible Requirements
- used for search engine indexing
- 1 billion pages per month
- HTML only
- ignore duplicate pages
- store page content up to 5 years

#### High-level design

![web_crawler](/assets/img/crawler.png)

##### Pick Seed URLs

- use seed URLs as a starting point for the crawl process
- based on locality
- based on topics

##### URL Frontier

component that stores URLs to be downloaded is called the URL Frontier: FIFO

##### HTML Downloader

download web pages

##### DNS resolver

URL must be translated into an IP address before downloading

##### Content Parser

parse the content of the web page, seperate component from crawling process

##### Content Seen Detector

Using Hash values to compare the content of the web page

##### Content Storage

disk + memory

##### URL Extractor

##### URL filter

##### URL Seen Detector

Bloom filter / Hash table

#### Way of Traverse

- BFS with FIFO
  - Most links from the same web page are linked back to the same host. If download web pages in parallel, servers will be flooded with requests. This is considered as “impolite”.
  - Standard BFS does not take the priority of a URL into consideration. Might need to prioritize URLs according to their page ranks, web traffic, update frequency, etc.

#### URL Frontier

A URL frontier is a data structure that stores URLs to be downloaded.

ensure politeness, URL prioritization, and freshness

- Politeness: avoid sending too many requests to the same host
  - A delay can be added between two download tasks
  - maintain a mapping from website hostnames to download (worker) threads

  ![mapping](/assets/img/mapping.png)

  - Queue router: It ensures that each queue (b1, b2, … bn) only contains URLs from the same host.
  - Mapping table: It maps each host to a queue.
  - FIFO queues b1, b2 to bn: Each queue contains URLs from the same host.
  - Queue selector: Each worker thread is mapped to a FIFO queue, and it only downloads URLs from that queue. The queue selection logic is done by the Queue selector.
  - Worker thread 1 to N. A worker thread downloads web pages one by one from the same host. A delay can be added between two download tasks.

- Priority:
  - PageRank, website traffic, update frequency
  - Prioritizer: It takes URLs as input and computes the priorities
  - Queue selector: Randomly choose a queue with a bias towards queues with higher priority
  - Queues with high priority are selected with higher probability
  ![priority](/assets/img/priority.png)
- Freshness:
  - Recrawl based on web pages’ update history.
  - Prioritize URLs and recrawl important pages first and more frequently.

#### HTML Downloader

##### Robots Exclusion Protocol

- robots.txt: standard used by websites to communicate with crawlers
- Before attempting to crawl a web site, a crawler should check its corresponding robots.txt first and follow its rules.

##### Performance optimization

- Distributed crawl: crawl jobs are distributed into multiple servers, URL space is partitioned into smaller pieces
  ![Distributed Downloader](/assets/img/Distributed_downloader.png)
- Cache DNS Resolver
  - Once a request to DNS is carried out by a crawler thread, other threads are blocked until the first request is completed. 
  - Maintaining our DNS cache to avoid calling DNS frequently is an effective technique for speed optimization. 
  - Our DNS cache keeps the domain name to IP address mapping and is updated periodically by cron jobs.
- Locality
  - Distribute crawl servers geographically. 
  - When crawl servers are closer to website hosts, crawlers experience faster download time.
- Short timeout
  - a maximal wait time is specified

#### Extensibility

![extension](/assets/img/extension.png)

- PNG Downloader module is plugged-in to download PNG files.
- Web Monitor module is added to monitor the web and prevent copyright and trademark infringements.


### Notification System

#### Main Components

- Different types of notifications
- Contact info gathering flow
- Notification sending/receiving flow

##### iOS push notification
- provider
  - Device token: a unique identifier for the device
  - Payload: a JSON dictionary that contains a notification’s payload.
- APNs
- iOS device

##### Android push notification

Instead of using APNs, Firebase Cloud Messaging (FCM) is commonly used to send push notifications to android devices.

##### SMS message

Third-party SMS services:
- Twilio
- Nexmo

##### Email

Third-party email services:
- SendGrid
- Amazon SES

##### Contact info gathering flow

![contact_gater](/assets/img/contact.png)
- for relational database, use *user* table, user_id as primary key, and *device* table, user_id as foreign key
- a user can have multiple devices

##### Notification sending flow

![notification](/assets/img/notification.png)
- service: a micro-service, a cron job, or a distributed system that triggers notification sending events