---
title: "Challenging Gossip Glomers"
date: 2024-01-28
author: "agao"
ShowToc: true
---

As with most things nowadays, I found [this](https://news.ycombinator.com/item?id=34897723) on HackerNews one day and bookmarked it somewhere in my list of TODO activities. I recently wanted to take a short break from working through [MIT's](https://pdos.csail.mit.edu/6.824/) distributed systems course, so I decided to take the time and try this out myself.

Gossip Glomers is a series of distributed systems challenges from [Fly.io](https://fly.io/dist-sys/) and [Kyle Kingsbury](https://aphyr.com/about). The challenge is meant to be language agnostic, but my implementation is in Go and the source code can be found [here](https://github.com/algao1/gossip-glomers).

I'll just be highlighting a few of the interesting tidbits from the challenges. And as always, **spoilers below**.

## Challenge 3: Broadcast

For this challenge, we need to implement a broadcast system that gossips messages between all nodes in the cluster. There are only 3 commands:

- `broadcast` broadcasts a value out to all nodes in the cluster
- `read` returns all values that a node has seen
- `topology` sets the neighbours of a node (who it can communicate with)

### Challenge 3b: Multi-Node Broadcast

The simplest approach here as mentioned would be to send a node's entire data on every message, however this is not practical in the real-world given that a message can be arbitrarily large.

Instead, for each node, we can send the message to a separate channel, which acts like a queue (one for each peer to broadcast), and batch them. This way we avoid having to send all messages at once, and reduces the number of messages we have to send.

However, we do need to take care to not gossip/rebroadcast any messages we've already seen to prevent an explosion of requests.

```go
// Only gossip if we haven't seen the message before.
if _, ok := s.messages[m]; !ok {
	for _, nChan := range s.neighbours {
		nChan <- m
	}
}
```

Here we batch and send the messages after a certain amount of time has passed, this is to ensure that messages are delivered on time and doesn't get stale. We can also add the option to batch and send after we've reached a certain number of messages, to keep the size of messages small.

```go
// Inside a separate goroutine to batch and send message.
// We have one for each peer n.
buffer := make([]int, 0)
ticker := time.NewTicker(time.Duration(150) * time.Millisecond)
for {
    select {
    case <-ticker.C:
        // ...
        // n is the peer we're sending the msg to.
        s.node.Send(n, msg)
    case m := <-nChan:
        buffer = append(buffer, m)
    }
}
```

### Challenge 3c: Fault-Tolerant Broadcast

To make it fault-tolerant, we only need to make one small addition to our previous solution, and that is to add retry logic to our `s.node.Send`. If we don't receive a message back within a certain timeframe, we can requeue all the messages and try again later.

```go
// Inside a separate goroutine to batch and send message.
buffer := make([]int, 0)
ticker := time.NewTicker(time.Duration(150) * time.Millisecond)
for {
    select {
    case <-ticker.C:
        // ...
        // n is the neighbour we're sending the msg to
        go func(batch []int) {
			ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
			defer cancel()
			_, err := s.node.SyncRPC(ctx, n, msg)
			if err != nil {
			    for _, m := range batch {
                    // Put the unsent batch back into the channel
					nChan <- m
				}
				return
			}
		}(batch)
    case m := <-nChan:
        buffer = append(buffer, m)
    }
}
```

## Challenge 4: Grow-Only Counter

In this challenge, we implement a stateless, grow-only counter using a _sequentially consistent_ key-value store. To do this, we make generous use of the atomic [**compare-and-swap**](https://en.wikipedia.org/wiki/Compare-and-swap) operation.

## Challenge 5: Kafka-Style Log

Here we implement a replicated log service similar to Kafka, and it has to support the following operations:

- `send` sends a message to be appended
- `poll` returns a set of logs starting from the provided offset
- `commit_offsets` informs the node of the offsets we have processed up to
- `list_committed_offsets` returns a map of committed offsets for a given set of logs

### Challenge 5b: Multi-Node Kafka-Style Log

There's two ways I've thought about tackling this problem:

1. The cluster runs leaderless, and we shard the logs amongst the nodes.
2. One node is assigned to be leader and all other nodes are followers and relay their requests to the leader.

I went with the first option since the challenge hinted at using a linearizable kv-store, but the second option more closely aligns with how Kafka looks like in reality.

#### Option 1: Leaderless

Each node is responsible (or owns) a subset of the logs, and we determine the owner by hashing the keys. When a node receives a write request for a key it doesn't own, it relays the request to the node that owns it.

```go
func (s *server) getOwner(key string) string {
	hash := fnv.New32a()
	hash.Write([]byte(key))
	i := int(hash.Sum32())
	if i < 0 {
		i *= -1
	}
	return s.node.NodeIDs()[i%len(s.node.NodeIDs())]
}
```

![kafka_leaderless](/blurb/img/kafka_leaderless.png)

We write our logs and committed offsets to shared kv-stores so that read requests (`poll` or `list_committed_offsets`) can read from the store even if it doesn't own the key, or directly from memory otherwise.

#### Option 2: Forward to Leader

In this case, we select one node to be the leader, and that node is solely responsible for executing all the commands. All other nodes are followers and relay both read and write requests to the leader.

This is relatively easy to implement, but has the problem of:

- Requiring too many messages (RPCs) for each operation.
- Single point of failure, if the leader goes down then we can neither read or write logs.

We can solve this by having the leader periodically update followers with new log entries and committed offsets, that way followers can serve as **read-replicas** and avoid overwhelming the leader with read requests. Though we would have to tolerate clients potentially reading stale information.
