---
title: "KV Store Performance Improvements"
date: 2025-01-20
author: "agao"
ShowToc: true
tags: ["database", "profiling"]
---

## Background

Almost two years ago I decided that I wanted to write my own [key-value store](/blurb/posts/kv_store) from scratch, however, I never got the chance to profile the code and make any performance improvements. Recently, I got an itch to look at some [flamegraphs](https://www.brendangregg.com/flamegraphs.html) so I decided that this would be the perfect opportunity to see if I can speed things up a bit. I recommend reading the previous post to get a better understanding of the work done here.

_Disclaimer: I'm neither an expert on databases or go profiling, so some of what I say may be completely incorrect._

As a baseline I just wrote a very simple benchmark that would test the write performance, uncompacted read performance, compaction speeds, and read performance.

```go
func main() {
    os.RemoveAll(TESTDIR)
    db, _ := lsm.NewLSMTree(
        TESTDIR,
        lsm.WithSparseness(8),
        lsm.WithMemTableSize(1024*1024*4),
        lsm.WithFlushPeriod(10*time.Second),
    )

    t := time.Now()
    b := make([]byte, 128)
    for i := range 1_000_000 {
        db.Put(fmt.Sprintf("key_%d", i), b)
    }
    fmt.Println("writing 1,000,000 entries", time.Since(t))

    db.Close()

    t = time.Now()
    for range 250_000 {
        db.Get(fmt.Sprintf("key_%d", rand.Intn(1_000_000)))
    }
    fmt.Println("reading 250,000 entries (uncompacted)", time.Since(t))

    t = time.Now()
    db.Compact()
    fmt.Println("compacting", time.Since(t))

    t = time.Now()
    for range 250_000 {
        db.Get(fmt.Sprintf("key_%d", rand.Intn(1_000_000)))
    }
    fmt.Println("reading 250,000 entries", time.Since(t))
}
```

The exact benchmarking code isn't super rigorous, but we can get a baseline of our performance with just that. Note that I only ran the benchmark a single time in each case, but you should really run it multiple times for a more fair comparison.

```console
writing 1,000,000 entries:              409.138409ms
reading 250,000 entries (uncompacted):  9.324291004s
compacting:                             15.239003143s
reading 250,000 entries:                3.340343153s
```

## Optimization 1: Buffered Writes

With profiling enabled, we can see this weird pattern in the CPU profile when we flush memtables to disk as SSTables (`LSMTree.FlushMemory` on the left):

![](/blurb/img/kv_store/profile1.png#center)

The most suspicious parts were the `file.Write` and `syscall.Write` stack frames on the top of the flamegraph to the left. And after looking around for a bit, I realized that was because the writer I was using wasn't buffering writes, and was directly writing to file (and thus performing a syscall) each time an entry was written. Yikes.

One simple change later to use a buffered writer, and we see a large improvement to the compaction speed (a **10.78x** reduction!). We also see a small improvement to the uncompacted read speeds, but I'm not exactly sure what could've caused it.

```go
dataPath := filepath.Join(sm.dir, fmt.Sprintf("lsm-%d.data", curCounter))
dataFile, err := os.OpenFile(dataPath, WR_FLAGS, 0644)
if err != nil {
    return fmt.Errorf("unable to flush: %w", err)
}
bw := bufferedWriter{bufio.NewWriterSize(dataFile, 1024*64)}
```

```console
writing 1,000,000 entries:              432.756532ms
reading 250,000 entries (uncompacted):  8.344874903s
compacting:                             1.413603421s
reading 250,000 entries:                3.228275608s
```

Nevertheless, onto the next flamegraph!

## Optimization 2: Sync Pool

![](/blurb/img/kv_store/profile2.png#center)

The next hotspot in the code seems to be coming from our reads, and in particular the `lsm.readChunk` function. This is called to read a segment (chunk) of entries from the data file, so we can iterate over it. There's quite a few `runtime.makeslice` calls which allocate memory (and is expensive), which tells me it's likely coming from something like `b := make([]byte, size)`.

We want to reduce this if possible, and one common approach to doing this is to use a `sync.Pool` which allows us to reuse allocated memory.

```go
sm := &SSTManager{
    dir:      dir,
    ssTables: make([][]SSTable, 1),
    bytesPool: sync.Pool{New: func() any {
        return new([]byte)
    }},
    sparseness: opts.sparseness,
    errorPct:   opts.errorPct,
    logger:     logger,
}

func (sm *SSTManager) findInSSTable(ss SSTable, key string) ([]byte, bool, error) {
    // ...

    offset, maxOffset := ss.Index.GetOffsets(key)
    if maxOffset < 0 {
        maxOffset = ss.FileSize
    }

    // Previously, this would just be chunkB := make([]byte, maxOffset - offset)

    chunkB := *sm.bytesPool.Get().(*[]byte)
    if cap(chunkB) < maxOffset-offset {
        chunkB = make([]byte, maxOffset-offset)
    }
    chunkB = chunkB[:maxOffset-offset]
    defer sm.bytesPool.Put(&chunkB)

    chunk, err := readChunkWithBuffer(ss.DataFile, offset, chunkB)

    // ...
}
```

Note that we may have to manually grow the returned byte slice since they are arbitrarily sized (we could get a reused smaller slice, or a brand new one).
There are no such guarantees when using a `sync.Pool`. I also didn't specify a default size for the byte pool since I didn't notice a significant improvement otherwise, but I might need to pick better sizes.

```console
writing 1,000,000 entries:              417.357875ms
reading 250,000 entries (uncompacted):  6.601543194s
compacting:                             1.727956858s
reading 250,000 entries:                2.478669755s
```

With this optimization, we see a pretty noticable improvement to both uncompacted and compacted reads, with a reduction of 26.4% and 30.2% respectively.
Let's see if we can squeeze a little more performance out of this.

## Optimization 3: Shift Entry Layout

![](/blurb/img/kv_store/profile3.png#center)

The DB also seems to be spending a significant amount of time checking whether an entry exists against the bloom filter.
But before tackling that, there's an easy optimization (slightly faster reads) we can do by rearranging the layout of each entry.
Currently each entry looks like this:

```console
+--------------------+-----+----------------------+-------+
| key size (8 bytes) | key | value size (8 bytes) | value |
+--------------------+-----+----------------------+-------+
```

So whenever we want to read an entry, we need to:

1. Read and parse 8 bytes to get the length of the key
2. Read and parse the key
3. Read and parse 8 bytes to get the length of the value
4. And lastly read and parse the value

To read a single entry, we need to perform 4 reads from the buffer (which does add up). We're not really concerned with the byte slice allocations, since they should be under 64KB and allocated onto the stack instead of the heap.

```go
func readKeyValue(reader io.Reader) (keyValue, int, error) {
    kb, kSize, err := readElement(reader)
    if err != nil {
        return keyValue{}, 0, fmt.Errorf("unable to read key: %w", err)
    }
    vb, vSize, err := readElement(reader)
    if err != nil {
        return keyValue{}, 0, fmt.Errorf("unable to read value: %w", err)
    }
    return keyValue{key: kb, value: vb}, kSize + vSize, nil
}

func readElement(reader io.Reader) ([]byte, int, error) {
    lb := make([]byte, 8)
    _, err := reader.Read(lb)
    if err != nil {
        return nil, 0, fmt.Errorf("unable to read length: %w", err)
    }

    l, _ := binary.Varint(lb)

    b := make([]byte, l)
    _, err = reader.Read(b)
    if err != nil {
        return nil, 0, fmt.Errorf("unable to read value: %w", err)
    }
    return b, int(8 + l), nil
}
```

Instead, we can frontload the key and value size data so that we only need to perform two reads (to the buffer) to read the entry.

```go
func readKeyVal(reader io.Reader) (keyValue, int, error) {
    lb := make([]byte, 16)
    _, err := reader.Read(lb)
    if err != nil {
        return keyValue{}, 0, fmt.Errorf("unable to read length: %w", err)
    }

    l1, _ := binary.Varint(lb[:8])
    l2, _ := binary.Varint(lb[8:])

    b := make([]byte, l1+l2)
    _, err = reader.Read(b)
    if err != nil {
        return keyValue{}, 0, fmt.Errorf("unable to read value: %w", err)
    }

    return keyValue{
        key:   b[:l1],
        value: b[l1:],
    }, int(l1 + l2), nil
}
```

With this, we again see a small improvement to reads performance (6.7% and 10.5% respectively):

```console
writing 1,000,000 entries:              406.321054ms
reading 250,000 entries (uncompacted):  6.184640688s
compacting:                             1.809599781s
reading 250,000 entries:                2.243038783s
```

## Optimization 4: Faster Bloom Filters

I was inspired to improve my bloom filter implementation after reading [this very informative blog post](https://save-buffer.github.io/bloom_filter.html).
I separated out the bloom filters into a separate package and ran a simple benchmark:

```console
❯ go test -bench=.
BenchmarkBloomFilterV1-4          442606              2700 ns/op
PASS
ok      crumbs/bloom    3.204s
```

One of the most expensive operations in a bloom filter is computing the _k_ different hash functions. We could improve this by using a fast hash function like `murmur3` or `xxhash`, but it would be far cheaper to just avoid doing all this work.

[This paper](https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf) recommends using two hash functions `h1(x)` and `h2(x)` to compute each of the _k_ hashes `g(x) = h1(x) + i * h2(x)`. This would reduce the number of hashes
we have to compute from _k_ to just 2. But we can do even better by leveraging the lower and upper 32-bits of a 64-bit hash as two separate hashes since they should be random and independent of each other.

```go
func cheatHash(h uint64, i int) uint32 {
    return uint32(h) + uint32(i)*uint32(h>>32)
}

func (bf *BloomFilterV2) Add(b []byte) {
    h := xxhash.Sum64(b)
    for i := range bf.K {
        pos := cheatHash(h, i) % uint32(len(bf.Bitset))
        bf.Bitset[pos] = true
    }
}

func (bf *BloomFilterV2) In(b []byte) bool {
    h := xxhash.Sum64(b)
    for i := range bf.K {
        pos := cheatHash(h, i) % uint32(len(bf.Bitset))
        if !bf.Bitset[pos] {
            return false
        }
    }
    return true
}
```

Using the new bloom filter, we get nearly a 10x improvement (although my naive implementation was not very good):

```console
❯ go test -bench=.
BenchmarkBloomFilterV1-4          446556              2656 ns/op
BenchmarkBloomFilterV2-4         4409061               272.9 ns/op
PASS
ok      crumbs/bloom    3.176s
```

This translate to a pretty substantial improvement to our performance across the board:

```console
writing 1,000,000 entries:              395.322549ms
reading 250,000 entries (uncompacted):  2.148468531s (3.07x)
compacting:                             1.171958263s (0.54x)
reading 250,000 entries:                2.18048968s
```

## Conclusion

I think there likely is still room for further optimization, mostly around reducing memory allocations (looking at the large chunks of `mallocgc` calls),
improving the filter performance (using a block bloom filter, or cuckoo filter), and using a skiplist.

But, I'm going to call it quits here as I've had enough fun looking at flamegraphs for a weekend.
