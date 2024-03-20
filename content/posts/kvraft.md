---
title: "KVRaft: Fault Tolerant Key Value Store Using Raft"
date: 2024-03-20
author: "agao"
ShowToc: true
---

This is a follow-up to my [previous post](/blurb/posts/raft) about writing Raft for MIT's 6.5840 (which I still need to finish). For the fourth lab, you need to build a fault-tolernace key/value store using the previous Raft library.

It supports only a few _key_ (pun intended) operations:

- `Put(key, value)`: replaces the value for a particular key in the database
- `Append(key, arg)`: appends arg to key's value (treating the existing value as an empty string if the key is non-existent)
- `Get(key)`: fetches the current value of the key (returning the empty string for non-existent keys)

The key here is that the operations need to be [**linearizable**](https://jepsen.io/consistency/models/linearizable), meaning that concurrent operations should behave as if they had executed in _some_ order, consistent with the real-time ordering of these operations.

## Basic Setup

The service consists of several key/value (`KVServer`) servers, each mapping to a Raft peer. Clients (`Clerks`) will send requests to the server associated with the leader Raft node (now referred to as the leader server).

![kvraft arch](/blurb/img/kvraft/kvraft_arch.png)

## Part A: No Snapshotting

To have a fault-tolerant key/value store we should only persist changes if a majority agrees on it. Luckily, Raft already does this for us so we can just append the `Put` and `Append` operations to our Raft log and process it when it has been applied.

However, we should also not serve stale data, meaning that a KVServer should not complete a `Get` request if it is not part of the majority. The simplest solution is to also write it to the log as before.

So, the life of any request generally goes like this:

1. Clerk sends a request to a KVServer
2. KVServer checks if it's the Raft leader, if not then reject the request
3. Otherwise, write the operation to the Raft log and register a callback
4. When the operation is applied, trigger the callback and return a response

Using the `Put/Append` request as an example, we have the following RPC handler:

```go
func (kv *KVServer) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
    op := Op{
        Key:    args.Key,
        Value:  args.Value,
        OpType: args.Op,
    }
    index, _, isLeader := kv.rf.Start(op)
    if !isLeader {
        reply.Err = ErrWrongLeader
        return
    }

    // Register callback channel.
    kv.mu.Lock()
    ch := make(chan Op, 1)
    kv.handlers[index] = ch
    kv.mu.Unlock()

    // Block until it has been processed, or we have timed out.
    select {
    case <-ch:
        _, isLeader = kv.rf.GetState()
        if !isLeader {
            reply.Err = ErrWrongLeader
            return
        }
    case <-time.After(TIMEOUT_DURATION):
        reply.Err = ErrTimedOut
        return
    }

    // Clean up the channels to prevent memory leaks.
    kv.mu.Lock()
    close(ch)
    delete(kv.handlers, index)
    kv.mu.Unlock()
}
```

Note that we need to factor in the case where the request times out (either due to network failure, or server failure) or if the server is no longer the leader. In which case, we want to fail and have the Clerk retry on another server.

The **trickiest** part here is deciding how and what messages to process. Specifically, we want to make sure that we only process each message once and reject any duplicate requests. This can happen if the leader fails after committing an entry, but hasn't had time to respond or if the response is lost.

We can ensure idempotency by maintaining a **duplicate table** of previously applied requests, and not performing the operation if an entry exists. This will require us to attach some metadata with each equest such as an unique `clientId` and `requestId`.

> We might be tempted to maintain a dupe table for each client, but that requires an enormous amount of space. Instead, we only need to maintain one entry per client since the IDs are monotonically increasing and only increases upon success.

Putting it all together, we end up with something like this:

```go
func (kv *KVServer) handleApplyCh() {
    for !kv.killed() {
        msg := <-kv.applyCh

        kv.mu.Lock()
        if msg.CommandValid {
            op := msg.Command.(Op)

            if requestId, ok := kv.dupeTable[op.ClientId]; !ok || op.RequestId > requestId {
                kv.dupeTable[op.ClientId] = op.RequestId
                if op.OpType == "Put" {
                    kv.values[op.Key] = op.Value
                } else if op.OpType == "Append" {
                    kv.values[op.Key] = kv.values[op.Key] + op.Value
                }
            }

            handler, ok := kv.handlers[msg.CommandIndex]
            if !ok {
                kv.dlog("No handlers found, index=%d", msg.CommandIndex)
                kv.mu.Unlock()
                continue
            }
            kv.mu.Unlock()

            handler <- op
            kv.dlog("Sent over handler, index=%d", msg.CommandIndex)
        }
    }
}
```

This seems simple enough, but the devil's in the detais, and I really struggled to get one thing right with this implementation. Here, we choose to always respond to the handler even if the request is a duplicate, **but what if didn't and ignored duplicate requests**?

> Well, the clients have no way of knowing that their requests are duplicates unless the KVServer tells them, so they will continue cycling through all the KVServers and block all subsequent requests. And we end up making no progress, not exactly ideal.

## Part B: With Snapshotting

TBD.
