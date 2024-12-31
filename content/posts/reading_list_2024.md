---
title: "A Year of Reading (2024)"
date: 2024-12-30
author: "agao"
---

I thought it'd be fun to revisit some of the best books and blogs I read this year.

- **Designing Data Intensive Applications** by Martin Kleppmann. One of the most recommended books for learning distributed systems. I think it gives a pretty comprehensive overview of everything someone should know. I would use it more as a reference book and read chapters based on interest.
- [**Bitcask**](https://riak.com/assets/bitcask-intro.pdf). This was one of the first papers I read this year. It's a pretty short read and the database is quite straightforward to implement. Would recommend this as the first paper for anyone looking to learn more about databases.
- [**Delta Lake**](https://www.vldb.org/pvldb/vol13/p3411-armbrust.pdf). A coworker recommended this to me, and it was very interesting to see the recent shift towards object-based storage like Warpstream. Though I didn't finish reading it fully.
- [**In Search of an Understandable Consensus Algorithm**](https://raft.github.io/raft.pdf). I mostly read this while implementing Raft for the [MIT 6.5840](https://pdos.csail.mit.edu/6.824/) labs. Also not a very difficult read, but you need to scrutinize the details to not mess up the implementation.
  - Also wanted to mention [this series](https://eli.thegreenplace.net/2020/implementing-raft-part-1-elections/#footnote-reference-1) of blog posts that helped me better understand Raft.
- [**Keeping CALM: When Distributed Consistency is Easy**](https://arxiv.org/pdf/1901.01930). The paper introduces the concept of monotonic programs, and how these programs have a consistent, coordination-free distributed implementation. I've never made this connection before, but it makes a lot of sense. Although these results don't really help us find implementations for these problems.
- [**Amazon's Dynamo**](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf). This was also a highly recommended paper alongside Google's Spanner. I just finished reading it and it definitely reinforced a lot of concepts and ideas for distributed systems, particularly about load distribution and availability vs. consistency trade-offs.
- [**Cinnamon: Using Century Old Tech to Build a Mean Load Shedder**](https://www.uber.com/en-CA/blog/cinnamon-using-century-old-tech-to-build-a-mean-load-shedder/) is a series of blog posts from Uber on building a load shedder for graceful degredation. I had a lot of fun reading this and trying to replicate this with my own [load shedder](https://github.com/algao1/crumbs/tree/master/load-shed).
- [**Google Prequal**](https://arxiv.org/abs/2312.10172). This was a fairly recent paper and the headline is that you should not be balancing load based on CPU or memory usage because they are a lagging indicator. Instead, you should be balancing on more real-time signals like requests-in-flight and latencies. Very unexpected read.
- **Deterministic Simulation Testing**. This is less of a single book or blog, and more of an overarching theme. I got super interested in the ability to deterministically replicate concurrency bugs.
  - Antithesis has some really [good blogs](https://antithesis.com/blog/) on everything deterministic state testing and beating video games with it.
  - The [sled](https://sled.rs/simulation.html) page gives a pretty good overview of how someone might implement DST for their systems.
- [**The One Billion Row Challenge in Go**](https://benhoyt.com/writings/go-1brc/). Was fun reading about the various performance optimizations you can do to process a billion rows in Go. No SIMD, but hoping to explore this more next year.
- [**Gossip Glomers**](https://fly.io/dist-sys/) is a series of distributed systems challenges from Fly.io. They were fun and a good introduction to distributed systems but I would have preferred something more difficult like the 6.5840 labs.

This was definitely not everything I read this year, but writing two sentence reviews for everything would take far too long. So, I've included some other good reads that I won't be writing more about (in no particular order).

- **Systems Performance** by Brendan Gregg. Just started reading but hoping to get through this next year.

- [Go Scheduler Control Theory](https://www.cockroachlabs.com/blog/rubbing-control-theory/)
- [The Tail at Scale](https://www.barroso.org/publications/TheTailAtScale.pdf)
- [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
