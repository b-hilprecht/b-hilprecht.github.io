In the [previous post]({{ "/2025/07/11/p-verified-log-3-introducing-failures.html" | relative_url }}), we introduced failures in our system and added logic to open a new segment when a write fails. This design already looked quite robust. But unfortunately, there is still a complex liveness bug. In this post, we will play detective and hunt this bug with a visualization tool. The full code of this blog post can as usual be found on [github.](https://github.com/b-hilprecht/verified-distributed-log/tree/main/03-liveness-bugfix)

In fact, the bug only appears after 1900+ schedules. Imagine how hard it would be to find such a bug with integration tests (not to speak of reproducing it). In a certain sequence of failures, the system will not make any more progress. It can be reproduced with the following command:

```bash
p check --fail-on-maxsteps --max-steps 1000 --seed 1948
```

## Debugging with PeasyViz

When a distributed system fails, traditional debugging approaches fall apart. There's no single call stack to examine, no obvious point of failure. Instead, we are faced with a complex choreography of messages, timeouts, and state transitions spread across multiple machines. Understanding what went wrong requires reconstructing the entire sequence of events that led to the problematic state.

This is where visualization tools like PeasyViz become invaluable. PeasyViz is a visualization tool that works with P to display system executions as interactive timelines, making it easier to understand complex distributed system behaviors. Instead of staring at log files or trying to mentally reconstruct message flows, we can see the entire system execution as an interactive timeline.

![Peasy overview](/assets/images/2025-07-10-p-verified-log/peasy_overview.png "Peasy overview")

At first glance, the visualization doesn't reveal anything obviously wrong. The system appears to be functioning—messages are flowing, components are responding. But the bug lies not in what's happening, but in what's not happening.

## Following the Breadcrumbs

Since we know this is a liveness violation (the system never terminates), we need to examine what state the system converges to. This is where the detective work begins.

Looking at the consumer, we see it's stuck trying to read log entries from the first segment. The log nodes keep responding with `READ_NO_MORE_ENTRIES`, indicating the segment is still open and has no more data available. But here's the puzzle: when we check the producer, it has already moved on to writing to the second segment.

![Peasy liveness problem](/assets/images/2025-07-10-p-verified-log/peasy_liveness.png "Peasy liveness problem")

This disconnect reveals the nature of our bug. The producer believes it's successfully closed the first segment and moved on. The consumer is waiting for that first segment to close so it can transition to reading from the second segment. But the log nodes think the first segment is still open. Someone's view of reality is wrong.

## The Smoking Gun

Digging deeper into the message flow reveals the smoking gun. The coordinator did send `endSegment` requests to the log nodes as designed, but here's where the failure scenario gets problematic: only a single log node received the message due to network failures. And that log node—the only one that knew the segment should be closed—crashed immediately after receiving the message.

![Peasy log node crash](/assets/images/2025-07-10-p-verified-log/peasy_log_node_crash.png "Peasy log node crash")

This creates a perfect storm: From the coordinator's perspective, it sent the close signal and can move on. From the producer's perspective, the segment transition succeeded and it can write to the new segment. But from the remaining healthy log nodes' perspective, the first segment is still open, so they won't serve data from it to consumers who are waiting for it to close.

The consumer is stuck in an infinite loop, waiting for a segment closure that will never come because the only nodes that could serve the data don't know the segment is supposed to be closed.

## The Fix: Persistence Over Perfection

The fix is quite simple: Instead of sending the `endSegment` message once and hoping it reaches all nodes, we need to ensure that all healthy log nodes eventually receive this critical information. The fix requires the coordinator to persistently retry the segment close operation until it receives acknowledgments from all healthy nodes:

```diff
@@ -35,12 +35,8 @@ machine Coordinator {
     var logNodes: set[LogNode];
     // List of segments per logId
     var logs: map[tLogId, seq[Segment]];
+    // We need to make sure the information that a segment is closed is eventually
+    // passed on to the log nodes. This is for retrying such requests.
+    var unacknowledgedEndSegments: set[tEndSegmentResponse];
     // Number of replicas for a log entry to be considered committed
     var numReplicas: int;
+    var timer: Timer;
 
     // Technicalities of the simulation. We keep track if producers and consumers are
     // done to shut down the remaining machines and avoid false-positive liveness issues.
@@ -56,7 +52,6 @@ machine Coordinator {
         entry (payload: (logNodes: set[LogNode], numReplicas: int)) {
             logNodes = payload.logNodes;
             numReplicas = payload.numReplicas;
+            timer = CreateTimer(this);
             goto Manage;
         }
     }
@@ -90,21 +85,11 @@ machine Coordinator {
             // how many entries in the segment can be considered committed. This then decides if
             // the last log entry will be returned to consumers or truncated.
             BroadcastToNodes(lastLogSegment.nodes, eEndSegment, (coordinator = this, logId = req.logId, segment = req.previousSegment, numEntries = req.previousSegmentNumEntries));
+            
+            // Keep track of which end segment requests need to be retried
+            if (sizeof(unacknowledgedEndSegments) == 0)
+                StartTimer(timer);
+            while (i < sizeof(lastLogSegment.nodes)) {
+                unacknowledgedEndSegments += ((node = lastLogSegment.nodes[i], logId = req.logId, segment = req.previousSegment, numEntries = req.previousSegmentNumEntries));
+                i = i + 1;
+            }
         }
 
         on eEndSegmentResponse do (req: tEndSegmentResponse) {
             assert req.logId in logs;
 
+            unacknowledgedEndSegments -= (req);
+            
             // it should concern the last log segment
             if (req.segment != sizeof(logs[req.logId])-1) {
                 return;
@@ -131,32 +116,8 @@ machine Coordinator {
             UnreliableSend(req.client, eSegmentStateResponse, (status = SEG_OPEN, nodes = segmentNodes, logId = req.logId, segment = req.segment));
         }
 
+        on eTimeOut do {
+            var newUnacknowledgedEndSegments: set[tEndSegmentResponse];
+            var segmentAck: tEndSegmentResponse;
+            var i: int;
+
+            // Retry unacknowledged end segments. Skip those directed to failed nodes
+            while (i < sizeof(unacknowledgedEndSegments)) {
+                segmentAck = unacknowledgedEndSegments[i];
+
+                if (segmentAck.node in failedNodes) {
+                    i = i + 1;
+                    continue;
+                }
+
+                UnreliableSend(segmentAck.node, eEndSegment, (coordinator = this, logId = segmentAck.logId, segment = segmentAck.segment, numEntries = segmentAck.numEntries));
+                newUnacknowledgedEndSegments += (segmentAck);
+                i = i + 1;
+            }
+            if (sizeof(newUnacknowledgedEndSegments) > 0)
+                StartTimer(timer);
+            unacknowledgedEndSegments = newUnacknowledgedEndSegments;
+        }
+
         on eShutDown do {
             var i: int;
+            send timer, halt;
             while (i < sizeof(logNodes)) {
                 send logNodes[i], eShutDown, logNodes[i];
                 i = i + 1;
```

The implementation tracks which nodes have acknowledged the segment close and retries for any nodes that haven't responded, while excluding nodes that are known to have failed.

This approach transforms a fire-and-forget message into a reliable delivery guarantee. It's more complex than the original design, but it eliminates the deadlock scenario by ensuring that segment state eventually converges across all healthy nodes.

## Testing the Fix

Now we can run the simulation again and see that the bug is fixed:

```bash
p compile && p check --fail-on-maxsteps --max-steps 100000 --seed 1 --timeout 1200 --explore

Compilation succeeded.
~~ [PTool]: Thanks for using P! ~~
.. Searching for a P compiled file locally in folder ./PGenerated/
.. Found a P compiled file: ./PGenerated/CSharp/net8.0/Log.dll
.. Checking ./PGenerated/CSharp/net8.0/Log.dll
.. Test case :: tcMultiReplica
... Checker is using 'random' strategy (seed:0).
..... Schedule #1
... Emitting coverage report:
..... Writing PCheckerOutput/BugFinding/Log.coverage.txt
..... Writing PCheckerOutput/BugFinding/Log.sci
... Checking statistics:
..... Found 0 bugs
```

The system now handles the failure scenario gracefully, with the coordinator persistently ensuring that all healthy nodes receive the segment close notification.

In the [next post]({{ "/2025/07/12/p-verified-log-5-single-producer-and-outlook.html" | relative_url }}), we will introduce a feature that is essential for using our design for real systems: the guarantee that only a single producer is active.

{% include 2025-07-10-series.md %}
