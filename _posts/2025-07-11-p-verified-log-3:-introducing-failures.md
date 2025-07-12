Our [happy path design]({{ "/2025/07/10/p-verified-log-2-modeling-the-happy-path.html" | relative_url }}) assumed we could write to the same three nodes forever. That assumption breaks down the moment any node becomes unavailable. In this case, we still want to be able to write log entries.

The Taurus approach suggests an elegant solution: when writes fail, start a new segment on different nodes. Simple in concept, but as we'll see, the implementation details hide surprising complexity. The full code of this blog post can as usual be found on [github.](https://github.com/b-hilprecht/verified-distributed-log/tree/main/02-failures)

## The Coordinator Service

When a write fails, two critical questions emerge: Which nodes should handle the next segment? How do we ensure only one segment is active at a time?

Our solution is to use external consensus: a simple coordinator that is responsible for assigning segments to nodes and managing the lifecycle of segments. Once a producer write fails, it will request a new segment from the coordinator. The coordinator will assign a new set of nodes to the segment and inform the producer about the new segment.

![log system design](/assets/images/2025-07-10-p-verified-log/log-1-taurus-overview.drawio.png "High-level log system design")

You might wonder: didn't we reject consensus protocols earlier for being too expensive? The key insight is that consensus overhead is acceptable for infrequent control plane operations. While we write log entries continuously (high frequency, performance-critical), segment transitions only happen during failures (low frequency, correctness-critical). Using a simple coordinator for segment management gives us the benefits of coordination where we need it without paying the consensus tax on our hot path. In this series, we will not model the consensus for the coordinator explicitly. We model the coordinator as reliable (no coordinator failures) but do model log node failures. In practice a coordinator implementation could simply use Paxos or a distributed KV store.

Let's thus look at how the Coordinator creates new segments:

```p
fun CreateSegment(producer: Producer): Segment {
    var chosenNode: LogNode;
    var potentialNodes: set[LogNode];
    var chosenNodes: set[LogNode];

    potentialNodes = logNodes;
    while (sizeof(chosenNodes) < numReplicas) {
        chosenNode = choose(potentialNodes);
        potentialNodes -= (chosenNode);
        chosenNodes += (chosenNode);
    }
    return (logNodes = chosenNodes, producer = producer);
}
```

## Producer Evolution: From Simple Replication to Segment Management

Our producer can no longer assume it writes to the same nodes forever. Instead, it must now handle segment lifecycles. In case of failure, it simply requests a new segment from the coordinator:

```p
state InitNewSegment {
    entry {
        UnreliableSend(coordinator, eNewSegment, 
            (client = this, logId = logId, previousSegment = currentSegmentId));
        StartTimer(timer);
    }

    on eNewSegmentResponse do (response: tNewSegmentResponse) {
        assert response.status == NEW_SEG_OK || response.status == NEW_SEG_ALREADY_APPENDED;
        
        currentSegmentId = response.newSegment;
        currentNodes = response.nodes;
        goto Produce;
    }

    on eTimeOut do {
        goto InitNewSegment;  // Retry segment creation
    }
}
```

During normal operation, the producer now watches for multiple failure signals:

```p
state Produce {
    on eAppendResponse do (response: tAppendResponse) {
        // Segment is closed due to failures
        if (response.status == APPEND_SEG_CLOSED && response.seqNum == currentEntry) {
            goto InitNewSegment;
        }

        // Successful write
        assert response.status == APPEND_OK;
        appendAcks += (response.logNode);
        if (sizeof(appendAcks) == sizeof(currentNodes)) {
            // All replicas acknowledged, move to next entry
            announce eLogEntryCommitted, (seqNum = response.seqNum, logId = logId);
            currentEntry = currentEntry + 1;
            goto Produce;
        }
    }

    on eTimeOut do {
        // Timeout indicates potential failures, start new segment
        appendAcks = default(set[LogNode]);
        goto InitNewSegment;
    }
}
```

This design handles failures gracefully: timeouts or explicit "segment closed" responses trigger segment transitions, allowing the system to continue making progress even when individual nodes fail.

## Consumer Adaptation: Finding Data Across Segments

Consumers face a more complex world too. They can no longer assume that a single log node contains all the data they need. Instead, they must discover which nodes hold each segment and handle segment transitions during reads:

```p
state FindSegmentNodes {
    entry {
        UnreliableSend(coordinator, eSegmentState, 
            (client = this, logId = logId, segment = currentSegmentId));
        StartTimer(timer);
    }

    on eSegmentStateResponse do (response: tSegmentStateResponse) {
        if (response.status == SEG_NOT_INIT) {
            goto FindSegmentNodes;  // Segment doesn't exist yet, retry
        }
        
        assert response.status == SEG_OPEN;
        currentNodes = response.nodes;
        goto Consume;
    }
}
```

The consumer must handle the case where segments don't exist yet—a timing issue that emerges when consumers try to read data that producers haven't finished writing.

When reading, consumers must gracefully handle segment boundaries:

```p
on eReadResponse do (response: tReadResponse) {
    if (response.status == READ_NEW_SEGMENT) {
        currentOffset = 0;
        currentSegmentId = currentSegmentId + 1;
        goto FindSegmentNodes;
    }

    if (response.status == READ_OK) {
        // Process the log entry and continue
        currentOffset = currentOffset + 1;
        goto Consume;
    }
}
```

This design ensures consumers can follow the log across segment boundaries, maintaining the illusion of a single continuous log despite the underlying segmented storage.

## LogNodes: Tracking What Matters

LogNodes evolve from simple append-only stores to sophisticated segment managers:

```p
// Status for each segment
var logStatus: map[tSegmentKey, tSegmentStatus];

on eAppendRequest do (req: tAppendRequest) {
    // Initialize segment if needed
    if (!(req.segmentKey in logEntries)) {
        logEntries[req.segmentKey] = default(seq[tLogEntry]);
        logStatus[req.segmentKey] = SEG_OPEN;
    }
    
    // Reject writes to closed segments
    if (logStatus[req.segmentKey] == SEG_CLOSED) {
        UnreliableSend(req.client, eAppendResponse, 
            (status = APPEND_SEG_CLOSED, segmentKey = req.segmentKey, 
             seqNum = req.seqNum, logNode = this));
        return;
    }

    // Accept write to open segment
    logEntries[req.segmentKey] += (sizeof(logEntries[req.segmentKey]), 
        (seqNum = req.seqNum, val = req.val));
    UnreliableSend(req.client, eAppendResponse, 
        (status = APPEND_OK, segmentKey = req.segmentKey, 
         seqNum = req.seqNum, logNode = this));
}
```

The segment status tracking ensures that once a segment is closed (due to failures), no further writes can be accepted.

## Testing What We Can't Predict

With failure-handling mechanisms in place, we need to verify they actually work under the chaos of real distributed systems. This is where P's systematic exploration becomes invaluable.

We create a failure injection framework that can introduce failures at any point during execution:

```p
type logConfig = (
    numReplicas: int,
    numLogEntries: int,
    numLogNodes: int,
    failParticipants: int
);

fun SetupLogSystem(config: logConfig) {
    // ... create producers, consumers, coordinator ...
    
    // Create failure injector if needed
    if(config.failParticipants > 0) {
        CreateFailureInjectorWithNotification(
            (nodes = logNodes, nFailures = config.failParticipants, 
             coordinator = coordinator));
    }
}
```

The failure injector systematically explores different failure scenarios: nodes crashing during segment creation, network partitions isolating subsets of nodes, and cascading failures where multiple nodes fail in sequence. Rather than hoping our manual tests cover the right edge cases, P explores all possible failure timings and interleavings. This systematic approach reveals bugs that would be nearly impossible to reproduce through traditional testing—like the "last entry problem" we'll encounter shortly.

## Evolved Formal Specifications

Our formal specifications must evolve because the introduction of segments fundamentally changes what "correctness" means. In our simple model, we only needed to ensure entries were replicated before being read. Now we must handle the complexity of entries spanning multiple segments, producers switching between node sets, and the ambiguity of what happens to in-flight entries during segment transitions. The specifications become more sophisticated to match the increased system complexity.

```p
spec CommitDurability observes eLogEntryCommitted, eLogEntryStored, eMonitor_CommitDurability {
    var logStoreEvents: map[tLogEntryCommitted, set[LogNode]];
    var requiredReplicas: int;

    on eLogEntryStored do (resp: tLogEntryStored) {
        var logEntry: tLogEntryCommitted;
        logEntry = (seqNum = resp.seqNum, logId = resp.logId);

        if (!(logEntry in logStoreEvents)) {
            logStoreEvents[logEntry] = default(set[LogNode]);
        }
        logStoreEvents[logEntry] += (resp.logNode);
    }

    on eLogEntryCommitted do (resp: tLogEntryCommitted) {
        assert resp in logStoreEvents, 
            format("Seq number {0} was not stored before being committed", resp.seqNum);

        assert sizeof(logStoreEvents[resp]) >= requiredReplicas,
            format("Seq number {0} was not stored to enough nodes before being committed", 
                   resp.seqNum);
    }
}
```

This specification ensures that even with segment transitions and node failures, we never commit a log entry until it's adequately replicated.

The ReadCommitted specification prevents inconsistent reads that are not yet considered committed.

```p
spec ReadCommitted observes eLogEntryCommitted, eReadResponse, eMonitor_ReadCommitted {
    var committedEntries: set[tLogEntryCommitted];

    on eLogEntryCommitted do (resp: tLogEntryCommitted) {
        committedEntries += (resp);
    }

    on eReadResponse do (resp: tReadResponse) {
        if (resp.status != read_OK) {
            return;
        }

        var logEntry: tLogEntryCommitted;
        logEntry = (seqNum = resp.seqNum, logId = resp.segmentKey.logId);

        assert logEntry in committedEntries, 
            format("Log entry {0} is not committed but was returned to consumer", logEntry);
    }
}
```

## The Last Entry Problem: When Ambiguity Bites

Running our enhanced model reveals a critical design flaw that's both subtle and devastating. When we switch segments due to failures, we can't determine whether the last entry of the previous segment should be considered committed.

Consider this scenario: A producer writes entry N to three nodes. Two nodes acknowledge successfully, but the third node fails before responding. The producer's timeout triggers a segment transition. Did entry N commit or not?

From the producer's perspective, it never received full acknowledgment—entry N shouldn't be committed. But from the successful nodes' perspective, they stored entry N and might serve it to consumers. It has no knowledge of whether the entry is also present on the other nodes.

## The Solution: Producer Authority

Rather than attempting complex coordination to resolve this ambiguity, we adopt a simpler approach: the producer, which has definitive knowledge of what it considers committed, informs the coordinator about exact commit boundaries.

When requesting a new segment, the producer includes precise commit information:

```p
state InitNewSegment {
    entry {
        UnreliableSend(coordinator, eNewSegment, 
            (client = this, logId = logId, previousSegment = currentSegmentId, 
             previousSegmentNumEntries = currentSegmentEntries));
        StartTimer(timer);
    }
    // ... rest of the state remains the same
}
```

Log nodes can now safely handle the last entry of closed segments:

```p
// Enhanced read handling for closed segments
numSegmentEntries = sizeof(logEntries[req.segmentKey]);
if (logStatus[req.segmentKey] == SEG_CLOSED && req.offset == numSegmentEntries - 1) {
    // For closed segments, the producer has told us exactly which entries are committed
    logEntry = logEntries[req.segmentKey][req.offset];
    UnreliableSend(req.client, eReadResponse, 
        (status = read_OK, segmentKey = req.segmentKey, offset = req.offset, 
         seqNum = logEntry.seqNum, val = logEntry.val));
    return;
}
```

When segments close, uncommitted entries are truncated based on the producer's authoritative count:

```p
on eEndSegment do (req: tEndSegmentRequest) {
    // Cut off all uncommitted entries
    while (sizeof(logEntries[segmentKey]) > req.numEntries) {
        logEntries[segmentKey] -= (sizeof(logEntries[segmentKey]) - 1);
    }
    assert sizeof(logEntries[segmentKey]) == req.numEntries;
    
    logStatus[segmentKey] = SEG_CLOSED;
}
```

This approach elegantly resolves the ambiguity by leveraging the producer's authoritative knowledge rather than attempting distributed consensus about commit status.

## The Road Ahead

The elegance of this solution lies in leveraging the producer's knowledge of commit status rather than attempting to reconstruct this information through complex distributed protocols.

We have now designed the basic ingredients of our log service under failures. Everything looks robust. However, running the P checker for a longer time we find another complex bug that we fix in the [next post]({{ "/2025/07/11/p-verified-log-4-debugging-with-peasyviz.html" | relative_url }}).

{% include 2025-07-10-series.md %}
