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

From *System Design Interview* by Alex Xu

- [K-V Store](#k-v-store)
  - [Design scopes](#design-scopes)
  - [CAP Theorem](#cap-theorem)
  - [System Components](#system-components)
- [Unique ID Generator](#unique-id-generator)
  - [Requirements](#requirements)
  - [Probable solutions](#probable-solutions)
    - [Multi-master replication](#multi-master-replication)
    - [UUID](#uuid)
    - [Ticket server](#ticket-server)
    - [Snowflake](#snowflake)
  - [Furhter problems](#furhter-problems)
- [URL Shortener](#url-shortener)
  - [API Design](#api-design)
    - [Redirecting](#redirecting)
    - [Shortening](#shortening)
  - [Possible process](#possible-process)
- [Web Crawler](#web-crawler)
  - [Possible Requirements](#possible-requirements)
  - [High-level design](#high-level-design)
    - [Pick Seed URLs](#pick-seed-urls)
    - [URL Frontier](#url-frontier)
    - [HTML Downloader](#html-downloader)
    - [DNS resolver](#dns-resolver)
    - [Content Parser](#content-parser)
    - [Content Seen Detector](#content-seen-detector)
    - [Content Storage](#content-storage)
    - [URL Extractor](#url-extractor)
    - [URL filter](#url-filter)
    - [URL Seen Detector](#url-seen-detector)
  - [Way of Traverse](#way-of-traverse)
  - [URL Frontier](#url-frontier-1)
  - [HTML Downloader](#html-downloader-1)
    - [Robots Exclusion Protocol](#robots-exclusion-protocol)
    - [Performance optimization](#performance-optimization)
  - [Extensibility](#extensibility)
- [Notification System](#notification-system)
  - [Main Components](#main-components)
    - [iOS push notification](#ios-push-notification)
    - [Android push notification](#android-push-notification)
    - [SMS message](#sms-message)
    - [Email](#email)
    - [Contact info gathering flow](#contact-info-gathering-flow)
    - [Notification sending flow](#notification-sending-flow)
  - [Reliability](#reliability)
  - [Addtional considerations](#addtional-considerations)
- [News Feed System](#news-feed-system)
  - [requirements](#requirements-1)
  - [high-level design](#high-level-design-1)
    - [Feed publishing](#feed-publishing)
    - [Newsfeed building](#newsfeed-building)
    - [Newsfeed cache](#newsfeed-cache)
- [CHAT SYSTEM](#chat-system)
  - [Requirements](#requirements-2)
  - [High-level design](#high-level-design-2)
    - [choice of network protocols](#choice-of-network-protocols)
    - [General Design](#general-design)
    - [Storage](#storage)
    - [Data Model](#data-model)
  - [Deep Dive](#deep-dive)
    - [Service discovery](#service-discovery)
    - [Message flows](#message-flows)
    - [Message synchronization across multiple devices](#message-synchronization-across-multiple-devices)


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
![better_notification](/assets/img/better_notification.png)
- service: a micro-service, a cron job, or a distributed system that triggers notification sending events
- Notification system
  - provides APIs 
  - basic validation of contact info
  - send notifications to message queues
  - Query the database or cache to fetch data needed to render a notification
- Third-party services
  - extensibility matters
  - service availability problems
- Cache & Database: User info, device info, notification templates, Settings
- Message queues: decouple & buffer
- Worker: simple logic, send notifications

#### Reliability

1. notification should be never-lost
   - database persistence: workers can hold notification log database
   - retry mechanism

2. exactly-once: can't ensure exactly-once delivery
   - dedupe mechanism: use Event ID to ensure idempotency

#### Addtional considerations

1. template reusing
2. notification settings
   - give users fine-grained control over notification settings
   - use setting table to store user settings
3. rate limiting
4. retry mechanism
5. security
   - appKey
   - appSecret
6. monitor queue
   - total number of queued notifications should be controlled
7. Event tracking
   - "bury the point"

![final_design](/assets/img/final_notification.png)


### News Feed System


#### requirements

- user can publish a post and see her friends’ posts on the news feed page
- feed is sorted mainly by reverse chronological order
- 10 million DAU
- contain media files, including both images and videos

#### high-level design

- Feed publishing: a user publishes a post, corresponding data is written into cache and database. A post is populated to her friends’ news feed.
- Newsfeed building: assume the news feed is built by aggregating friends’ posts in *reverse chronological* order.

##### Feed publishing

![feed](/assets/img/feed_pub.png)

- Post service: persist post in the database and cache.
- Fanout service: push new content to friends’ news feed. Newsfeed data is stored in the cache for fast retrieval.
- Notification service: inform friends that new content is available and send out push notifications.

![feed_final](/assets/img/feed_final.png)

- fanout service
  - fanout on write (also called push model): A new post is delivered to friends’ cache immediately after it is published
    - Pros: 
      - news feed is generated in real-time and can be pushed to friends immediately
      - Fetching news feed is fast
    - Cons:
      - hotkey problem: a user has a large number of friends, fetching the friend list and generating news feeds for all of them are slow and time consuming
      - For inactive users or those rarely log in, pre-computing news feeds waste computing resources
  - fanout on read (also called pull model): The news feed is generated during read time. This is an on-demand model.
    - Pros:
      - For inactive users or those who rarely log in, fanout on read works better because it will not waste computing resources on them
      - Data is not pushed to friends so there is no hotkey problem
    - Cons:
      - Fetching the news feed is slow as the news feed is not pre-computed
  - Pratical Solution: Hybrid model
    - push model for the majority of users
    - pull model for celebrity users or those with a large number of friends
    -  Consistent hashing is a useful technique to mitigate the hotkey problem as it helps to distribute requests/data more evenly

##### Newsfeed building

![newsfeed](/assets/img/newsfeed.png)

- Newsfeed service: news feed service fetches news feed from the cache.
- Newsfeed cache: store news feed IDs needed to render the news feed.

![retrieval](/assets/img/retrieval.png)

##### Newsfeed cache

like `<post_id, user_id>` mapping table

### CHAT SYSTEM

#### Requirements

- support both 1 on 1 and group chat
- support 50 million daily active users (DAU)
- group member limit: maximum of 100 people
- only supports text messages
- text length should be less than 100,000 characters long
- chat history persistence
- Multiple device support

#### High-level design

Use basic client-server architecture:

- Server
  - receives messages
  - relays messages to the intended recipients
  - store if not online

##### choice of network protocols

1. Sender side

Uses HTTP to initially send messages to the server.
- use keep-alive header
  - maintain a persistent connection
  - reduces the number of TCP handshakes

2. Receiver side

- Polling: Client periodically asks the server if there are any new messages.
  - costly
  - consume server resources and bandwidth
- Long polling: a client holds the connection open until there are actually new messages available or a timeout threshold has been reached
  - Sender and receiver may not connect to the same chat server. HTTP based servers are usually stateless. If you use round robin for load balancing, the server that receives the message might not have a long-polling connection with the client who receives the message.
  - A server has no good way to tell if a client is disconnected.
  - It is inefficient. If a user does not chat much, long polling still makes periodic connections after timeouts
  - WebSocket: bi-directional and persistent, port 80 or 443
    - starts its life as a HTTP connection and could be “upgraded” via some well-defined handshake to a WebSocket connection
    - ![websocket](/assets/img/ws.png)

##### General Design

Note: everything else does not have to be WebSocket (sign up, login, user profile, etc)

![general_design_for_chat](/assets/img/chat_design.png)

- service discovery: give the client a list of DNS host names of chat servers that the client could connect to
- chat service: only stateful service
  - stateful because each client maintains a persistent network connection to a chat server

adjusted version without single-server problem:

![adjusted_version](/assets/img/adjusted_version.png)
- chat servers: message sending/receiving
- presence servers: online/offline status
- API servers: everything stateless
  - user login, signup, change profile, etc
- KV store: chat history, for scenarios like offline user comes online

##### Storage

Relational or NOSQL database?

- check data types
  - generic data: user profile, setting, user friends list
    - relational database 
  - chat history data: 
    - enormous
    - only recent data is frequently accessed
    - users might use features require random access (search, view your mentions, jump to specific messages)
    - Read Write Ratio (R/W) is 1:1 for 1 on 1 chat
    - Use KV store for chat history
      - easy horizontal scaling
      - very low latency to access data
      - Relational databases do not handle long tail of data well
      - Most existing services use KV store for chat history

##### Data Model

![message](/assets/img/message.png)

![group_message](/assets/img/group_message.png)

explanation about message ID:
- IDs must be unique
- IDs should be sortable by time, meaning new rows have higher IDs than old ones

approach:
- global ID generation: snowflake
- local sequence number generator: because maintaining message sequence within one-on-one channel or a group channel is sufficient

#### Deep Dive

##### Service discovery

recommend the best chat server for a client
- Apache Zookeeper
  - use znodes to store chat server IP addresses
  - each chat server registers its IP address to Zookeeper
  - check more details about zookeeper in *Common Sense for Distributed System*

![explanation_server_discovery](/assets/img/server_discovery.png)

##### Message flows

![message_flow](/assets/img/message_flow.png)

##### Message synchronization across multiple devices


