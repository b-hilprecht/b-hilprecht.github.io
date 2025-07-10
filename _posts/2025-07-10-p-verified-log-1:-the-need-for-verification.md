Cloud databases face a fundamental challenge: how to remain available and durable under node failures? Modern cloud databases approach this by separating two concerns that used to be tightly coupled: compute and storage. The database engine becomes stateless, while the write-ahead log gets replicated across multiple nodes to guarantee durability. If a database server dies, another one can pick up exactly where it left off by reading from the replicated log.

Distributed log services are thus at the heart of cloud databases. In this blog post, we will explain some drawbacks of the predominant design for distributed logs to motivate a new elegant design. We will also explain why it is necessary to verify this design with formal methods.

## The Consensus Tax

Most systems today, including [Amazon Aurora](https://assets.amazon.science/dc/2b/4ef2b89649f9a393d37d3e042f4e/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases.pdf), use consensus protocols like Paxos to manage log replication.

While this design works in practice, consensus protocols come with significant overhead. They exhibit high write amplification—if we want to guarantee three copies of our data, we need at least five nodes since we need a quorum. That's 67% (5 nodes instead of 3) more infrastructure than strictly needed. They also require at least two network roundtrips: from client to leader, then from leader to follower nodes and back.

## Taurus: Being frugal in the cloud

This is where the [Taurus system](https://arxiv.org/abs/2412.02792) offers an intriguing alternative: what if we just picked three nodes from a cluster and wrote our data to them? When a write fails, pick three different nodes and continue. No complex consensus protocol, no quorum calculations, no multi-round coordination.

![log system design](/assets/images/2025-07-10-p-verified-log/log-1-taurus-overview.drawio.png "High-level log system design")

The system architecture is thus straightforward. Producers write to a log segment replicated to multiple log nodes until a write fails and a new segment is started. The consumers simply read from the segments in order. The coordination is done using external consensus, e.g., a control plane.

This approach has some appealing properties:

*   **You only need three nodes for three copies of data** — no more paying the consensus tax
    
*   **Network round trips are minimized** — direct communication with log nodes eliminates leader-follower coordination
    
*   **High availability comes naturally** — as long as any three nodes in your cluster are healthy, we can append log entries

*   **Readers need only a single log node** - we can read the log entries from any node similar to consensus protocols

In a large cluster, the probability of not finding three healthy nodes is vanishingly small. For more details, [Alex Miller's blog](https://transactional.blog/talk/enough-with-all-the-raft) has an excellent description and a more in-depth comparison with quorum and consensus algorithms.

## But Does It Actually Work?

While these properties make Taurus attractive, the seemingly simple design introduces complex failure scenarios. What happens when network partitions occur? How do we handle partial writes (log entries only made it to a subset of log nodes)? What about consistency guarantees when switching between different sets of nodes?

Each of these scenarios can lead to subtle correctness bugs that only manifest under specific failure conditions—exactly the kind of bugs that are very hard to catch with traditional testing.

## Why Verification Matters

To address these challenges, we will use formal verification to ensure the correctness of our log system design.

The [P language](https://github.com/p-org/P) is designed specifically for modeling and verifying distributed systems. It's not just a research tool—AWS uses P extensively to verify real systems. Parts of S3's strong consistency guarantees were recently verified using P, demonstrating its practical value for real-world systems.

Formal verification allows us to systematically explore all possible system states and transitions between them. We can model failure scenarios that might be missed in traditional testing and prove correctness properties like safety (nothing bad happens) and liveness (something good eventually happens).

More importantly, verification catches the subtle bugs that lurk in edge cases—the ones that only show up when everything is going wrong at once.

## What We're Building

The Taurus paper provides a compelling vision but leaves many implementation details as open questions. That gives us an opportunity: we can design a complete log system inspired by Taurus and use formal verification to ensure it handles all failure scenarios correctly. I am not aware of any more details of the Taurus system beyond the ones provided in the paper. Hence, in this blog we will thus just design a system inspired by the idea.

In this series, we'll build and verify our system model step by step. We'll start by modeling the basic replication protocol assuming perfect networks and no failures. Then we'll add the complexity that makes distributed systems interesting: node failures, network partitions. Along the way we will become familiar with the P language including debugging tools.

The next post will dive into our first P model. We'll design the basic protocol for writing log entries to multiple nodes and verify that it maintains our safety properties even in this simplified world. From there, we'll add failures until we have a complete, verified system.
