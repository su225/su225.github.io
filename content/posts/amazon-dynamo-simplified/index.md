---
title: "Amazon Dynamo - simplified"
date: 2022-01-12T20:00:17+05:30
draft: true
tags: [distributed-systems, amazon]
---

[Link to the paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

From the abstract of the paper (emphasis-mine)
> A **highly available key-value storage system** that some of Amazon’s
core services use to provide an **“always-on”** experience. To
achieve this level of availability, Dynamo **sacrifices consistency**
under certain failure scenarios. It makes extensive use of object
versioning and application-assisted conflict resolution

### Objectives
1. Incremental and high scalability, read and write throughput
2. High availability and (Ideally) always writeable store even under events like network partitions
3. Low latency to meet strict SLA (p99.9 < 300ms for peak client load of 500 req per second)

### Issues with data-stores before Dynamo
What was the problem with the then existing data-stores? (mostly traditional RDBMS at that time)
1. Hard to partition data and scale across multiple nodes
2. Even when partitioning is possible, consistency is preferred. According to the CAP theorem,
   under network partition, even assuming eventual consistency, the store would not be writeable
   for one side of the partition.
3. Complex query processing adds to latencies. For most workloads, primary key access and
   eventual consistency was good enough.

### Assumptions
1. **Primary key access** - only `get(key)` and `put(key,value)` is good enough
2. **Eventual consistency** - that is, the value read could be stale for some time
3. **Weak consistency and write conflict** - applications might have to resolve conflicts of the value
4. Lack of strong isolation levels
5. Same administrative domain
6. Node failures are a norm (common for distributed systems)

### Design
1. **Data partitioning** - There is a limit to scaling up. So scale out architecture is adopted with
   the data (key-value pairs) partitioned across multiple nodes with a consistent hashing scheme with
   each node assigned multiple "tokens" corresponding to different positions in the ring.

2. **Data replication** - To ensure high availability in the face of node failures, the data is replicated
   across multiple nodes in multiple data-centers connected by high-speed links. Multiple copies introduces
   the good old problem of keeping replicas in-sync.
   1. **Ordering writes** - Dynamo does not do that. It only provides [causal consistency](https://jepsen.io/consistency/models/causal).
      Any node in the preference list can act as a coordinator. This improves the write throughput of the system.
      Assuming the nodes are spread well-enough, the client can reach out to its "nearest" node responsible for
      the partition and send the writes to it. This reduces the latency as not all nodes need to acknowledge writes
    
   2. **Handling write conflicts** - The problem with allowing any node in the preference list to be a coordinator is
      that when there are concurrent updates to the same key, the result is non-deterministic. There is no order. The
      conflict has to be resolved either by the application or by some simple strategies like "last-write wins". 
      
      _Why this decision?_ In case it has to be resolved by the application, then it makes the life of the application
      developer hard. The alternative is to elect a leader and order writes through it. Problem is - writing would
      not be possible on one side of the network partition and this does not satisfy the high availability and always
      writeable requirement. So writing multiple versions (even on both sides of the network partition) is allowed.
  
   3. **Versioning and causality** - Dynamo uses hybrid timestamp scheme (vector timestamp and clock-time). This is
      necessary to determine conflicts and concurrent write scenarios.

3. **Decentralized metadata maintenance** - Because centralized metadata stores, although scalable for very
   large clusters, become another component whose availability has to be maintained in the face of network
   partitions, data-center failures. Decentralized peer-to-peer architecture is resilient to these kind of
   failures.
   1. **Membership maintenance** - Members are added and removed explicitly by an administrator.
   2. **Failure detection** - While failures are detected and alerted, it does not result in membership
      changes. Unreachability does not result in the departure of the node from the ring.
   3. **Gossip protocol** to (probabilistically) broadcast the metadata/partition assignment to each node
   4. For each partition, there is a **preference list** consisting of the ordered list of nodes where the
      data needs to be placed. Each key is replicated `N` times. The number of unique physical nodes in the
      preference list would be more than `N` to account for node failures
