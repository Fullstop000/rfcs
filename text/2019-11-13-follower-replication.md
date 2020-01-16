# Follower Replication

## Summary

This RFC introduces a new mechanism into Raft protocol which allows a follower to send raft logs and snapshots to other followers. The target of this feature is to reduce network transmission cost between different data centers in Log Replication.

## Motivation

In the origin Raft protocol, a follower can only receive raft logs and snapshots from its leader, which could be insufficient in some situations. For example, when a Raft group is distributed in different data centers, log replication between a new node and the leader could be expensive if they are in different data centers. In this case, internal follower-to-follower transfer in one data center can be far more efficient than the traditionally stipulated leader-to-follower transfer. And in a cluster with massive nodes, the leader can have lower system load due to the less operations in network transmission.

## Detailed design

There are four key concepts of design:

- Every peer in a Raft group is associated with a `group_id`.
- Follower-to-follower data transfer is only allowed for peers in same group.
- Peers report their `group_id` to their leader in Raft messages.
- Leader is able to ask a follower to replicate data to other followers.

### The Group and Groups

We introduce a new concept *group* that represents all the raft nodes in a data center and any node which belongs to the group can be called a *group member*. Every raft node should have a new configuration called *Groups* which contains all the groups as the name is. For example, in a 2DC based raft cluster with 5 nodes ABCDE where A, B, and C are at DC1 and  DE are at DC2, the `Groups` might be organized like:

```
Group1: [A, B, C]
Group2: [D, E]
```

The leader must be easy to know whether or not nodes belong to the same group or get all the group members. Since `Groups` is a configuration that will never be persistent in Storage and is volatile, a raft node exposes the ability to modify it on the fly with the benefit of flexibility on the upper layer.

### The Delegate and Commission

As the `Groups` is configured, the leader is able to choose a group member as a *delegate* to send entries to one or the rest group members in Log Replication, which is called Follower Replication. To implement Follower Replication, we basically need three main steps:

1. The leader picks a group member as a delegate of the group.
2. The leader tells the delegate the name list of other peers in the same group with the delegate.
3. the delegate replicates new appended entries to those peers.

Here is a diagram showing how Follower Replication works:

```
                                        +-----------+
                                        |           |
       +--------------------------------+     C     |
       |           MsgAppendResp        |           |
       |                                +-----^-----+
       |                                      |
       |                                      |  MsgAppend(e[4,5])
       |                                      |
+------v-----+                          +-----+-----+
|            |                          |           |
|   leader   +-------------------------->     B     | delegate
|            |   MsgAppend {            |           |
+------^-----+     e[2,5]               +-----+-----+
       |           BcastTargets [C, D]        |
       |         }                            |  MsgAppend(e[3,5])
       |                                      |
       |                                +-----v-----+
       |                                |           |
       +--------------------------------+     D     |
                   MsgAppendResp        |           |
                                        +-----------+

```

Note:

- `e[x, y]` stands for all the entries within an index range [x, y] (both inclusive)
- `BcastTargets` is a name list of delegated peers.

#### Choose a Delegate

Since a delegate is responsible to send entries to other group members, it must contain entries as more as possible to complete the commission. The leader follows a bunch of rules to choose a delegate of a group:

1. If all the members are requiring snapshots, choose the delegate randomly.
2. Among the members who are requiring new entries, choose the node satisfies conditions below:

    1. The progress has the maximal `match_index`.

3. If no delegate is picked, the leader does Log Replication itself. Especially, if a group contains the leader it self, no delegate will be set. And the feature of enabling choosing delegate of such 'leader group' can be controlled by upper layer.

If the delegate of a group is determined, it’ll be cached in memory for later usage until Groups configuration is changed or the delegate is evicted by rejection message. And the flow control should be valid for any cached delegate.

#### How delegate does replication

We can add a new field into `Message`: `BcastTargets`, which is a list of peer IDs that the message receiver (the delegate) needs to replicate entries to. When a peer become delegate first time, it doesn't know followers' progress, so it appends its last log entry to other peers. After it receives `MsgAppendResponse` from those peers, it can update their progresses.

You can just treat a commission as the metadata of a MsgAppend or MsgSnapshot since the `last_index` and `log_term` are set by the leader according to its progress set.

Peers can know `MsgAppend` comes from a delegate or its leader, because an another field `delegate` is added in `Message`. If a peer receives a `MsgAppend` from a delegate, it needs to response both its leader and the delegate, so that both its leader and delegate can update progress for it.

## Drawbacks

1. Committing speed could be slow if `quorum` must involve peers in different data center.
2. When handle a Raft message, a hashmap query on `map[peer_id]group_id` is introduced.

## Alternatives

## Unresolved questions
