---
title: "Raft: Simple but Not Easy"
date: 2024-02-08
author: "agao"
ShowToc: true
draft: true
---

This post highlights my weeks long journey of implementing Raft, following from [MIT's 6.5840](https://pdos.csail.mit.edu/6.824/) Distributed Systems course. Specifically, this post covers some of what's required to implement [Lab 3](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html), a distributed consensus algorithm for managing a replicated [log](https://en.wikipedia.org/wiki/Transaction_log).

Despite the authors' claims that Raft is _simpler and more understandable than other algorithms_, it is still very complicated, and requires some careful consideration. Here I'm assuming that the reader has had a chance to read the [**Raft paper**](https://raft.github.io/raft.pdf), and maybe some of the provided resources.

The complete implementation can be found [here](https://github.com/algao1/6.5840), but I also highly recommend checking out [this series of posts](https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/).

## Resources

Here's a collection of resources that I found helpful, roughly in the order that I recommend reading them.

- [Raft Paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)
- [MIT 6.5840 (2024)](https://pdos.csail.mit.edu/6.824/)
  - [Student's Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)
  - [Raft Notes](https://pdos.csail.mit.edu/6.824/notes/l-raft-QA.txt)
- [Implementing Raft, by Eli Bendersky](https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/)
  - Overall great resource for walking through implementing the majority of Raft. Only thing it doesn't cover are snapshotting and some other optimizations.
- [Debugging by Pretty Printing](https://blog.josejg.com/debugging-pretty/)
  - I would recommend taking a look at this before starting. Having the proper tooling for logging and test eval makes the process a lot more enjoyable.

## Part A: Leader Election

For the first part of the lab, we need to implement **leader election** and **heartbeats**, so that leaders can be elected, and allow followers to take over if there are failures.

To begin, there are three states a Raft node can be in: Follower, Candidate, and Leader.

```go
type RaftState int
const (
  Follower RaftState = iota
  Candidate
  Leader
)
```

Each state has some associated work that needs to be done periodically, and I've separated these into their respective `tickFunc`s.

```go
func (rf *Raft) ticker() {
  for !rf.killed() {
    rf.mu.Lock()
    rf.tickFunc()
    rf.mu.Unlock()
    time.Sleep(time.Duration(10) * time.Millisecond)
  }
}
```

Nodes will spend the majority of their time as followers, and will remain as followers so long as they receive periodic communications from the Raft leader. In particular, **this timer is reset whenever**:

- We get an `AppendEntries` RPC (we will see this later) from the _current leader_
- We are starting an election
- We are granting a vote to another peer

Otherwise, the follower will trigger an election and becomes a candidate. When an election is triggered, the candidate will vote for itself, increment its current term, and immediately solicit votes from other nodes.

```go
func (rf *Raft) tickFollower() {
  if !rf.killed() && time.Since(rf.lastElectionEvent) > rf.electionTimeout() {
    rf.becomeCandidate()
  }
}

func (rf *Raft) becomeCandidate() {
  rf.currentState = Candidate
  rf.currentTerm++
  rf.votedFor = rf.me
  rf.lastElectionEvent = time.Now()
  rf.tickFunc = rf.tickCandidate
  go func() {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    rf.candidateRequestVotes()
  }()
}
```

This is done by sending `RequestVote` RPCs to peers to solicit votes, and these votes are given on a first-come, first-served basis for _each term_. Each message is sent over a separate goroutine since we don't want to block progress while waiting for a reply. Once a candidate has a majority of the votes, it can then transition to a leader.

```go
// Inside candidateRequestVotes() ...
for peerId := range rf.peers {
  go func(peerId int) {
    args := RequestVoteArgs{
      Term:        savedCurrentTerm,
      CandidateId: rf.me,
    }
    var reply RequestVoteReply
    if !rf.sendRequestVote(peerId, &args, &reply) {
      return
    }

    rf.mu.Lock()
    defer rf.mu.Unlock()

    if rf.currentState != Candidate || reply.Term > savedCurrentTerm {
      return
    }

    if reply.Term == rf.currentTerm {
      if reply.VoteGranted {
        votesReceived++
        if votesReceived*2 > len(rf.peers) {
          rf.becomeLeader()
          return
        }
      }
    }
  }(peerId)
}
```

One thing to note here is the check for `rf.currentState != Candidate`. This is because a candidate or leader can transition back to a follower if it receives an RPC with a term that is higher than its current term.

Lastly, once a node becomes a leader, it must actively assert and maintain leadership. This is done by sending out heartbeats periodically, which is just an empty `AppendEntries` RPC with no entries to append.

```go
func (rf *Raft) leaderAppendEntries() {
  rf.lastHeartbeatEvent = time.Now()
  savedCurrentTerm := rf.currentTerm

  for peerId := range rf.peers {
    if peerId == rf.me {
      continue
    }
    go func(peerId int) {
      args := AppendEntriesArgs{
        Term:     savedCurrentTerm,
        LeaderId: rf.me,
      }
      var reply AppendEntriesReply
      if !rf.sendAppendEntries(peerId, &args, &reply) {
        return
      }

      rf.mu.Lock()
      defer rf.mu.Unlock()

      if rf.currentState != Leader {
        rf.dlog("... while waiting for AppendEntries reply, changed state=%v", rf.currentState)
        return
      }

      if reply.Term > savedCurrentTerm {
        rf.dlog("... skipping AppendEntries reply since term out of date")
        rf.becomeFollower(reply.Term)
      }
    }(peerId)
  }
}
```

_The two receiver implementations for `RequestVote` and `AppendEntries` can be found in the source code._

**Another thing** to be wary of here is the timing of leader elections and heartbeat events. The paper recommends that heartbeat timeouts should be an order of magnitude smaller than election timeouts, so that leaders can reliably send heartbeats to prevent followers from starting elections.

The testing harness limits us to tens of heartbeats per second, so we need to pick a reasonable election timeout to match. Personally, I've found timeouts of `100ms` and `300 + [0, 300)ms` to work.

## Part B: Log Replication

WIP.
