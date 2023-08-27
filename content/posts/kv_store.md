---
title: "Implementing a Key-Value Store From Scratch"
date: 2023-08-23
author: "agao"
ShowToc: true
---

A few months ago I was inspired [this post](https://artem.krylysov.com/blog/2023/04/19/how-rocksdb-works/) on how RocksDB works and also [DDIA](https://dataintensive.net/), and decided that it would be a fun exercise to implement a database myself. This post is more of a documentation of how I built my own key-value store, but I still hope to give an overview of how they work, and provide enough details for someone to implement one themselves. 

The database and code snippets are written in Go, and the source code can be found [here](https://github.com/algao1/crumbs/tree/master/dbs/lsm).

## Log Structured Merge Tree

For our implementation, we'll be using a log structured merge tree (or LSM tree). It is a data structure that is comprised of two other data structures, **memtables** which exists strictly in memory, and **SSTables** which exists strictly on disk. The memtables are kept in the first layer of the tree, sorted in chronological order. Similarly, the SSTables are divided into different layers from newest to oldest.

![LSM Tree](/blurb/img/lsm_tree.png)

This setup allows all our writes to be sequential (which is really fast!), and have decent read speeds.

```go
// lsm_tree.go
type LSMTree struct {
    tables []Memtable
    stm    *SSTManager
    // ...
}
```


## Writes

Writes (or inserts) to the database are appended directly to the newest memtable, and once that memtable reaches a certain threshold, it is rotated out for a new one. Every memtable except the latest one is immutable and read-only.

A memtable is just an interface, and any structure ([AA tree](https://user.it.uu.se/~arnea/ps/simp.pdf), red-black tree, skiplist, etc.) that implements finding/inserting a key-value pair (kv-pair) and listing all kv-pairs in order, is sufficient.

```go
// lsm_tree.go
type Memtable interface {
    Find(key string) ([]byte, bool)
    Insert(key string, val []byte)
    Traverse(f func(k string, v []byte))
}
```

```go
// lsm_tree.go
func (lt *LSMTree) Put(key string, val []byte) {
    curTable := lt.tables[len(lt.tables)-1]
    curTable.Insert(key, val)

    if curTable.Size() > lt.memTableSize {
        curTable = NewAATree()
        lt.tables = append(lt.tables, curTable)
    }
}
```

If the number of memtables exceeds a given threshold, then it is persisted to disk as a SSTable. This can be done in many different ways, but for simplicity, I've opted to have a background process that periodically flushes extra tables to disk.

### Persisting Memtables

We persist our memtable into a SSTable, which is just a subset of key-value pairs (those from the memtable) in *some* sorted order. Since it is stored on disk, we need to encode it in a format that is compact, and easy to retrieve.

For our purposes, since we want to support variable-lengthed keys, we'll first encode the length of the key (which is fixed to 8 bytes), and then the actual key.

```go
// data_file.go
func flushBytes(file *os.File, b []byte) (int, error) {
    lb := make([]byte, 8)
    binary.PutVarint(lb, int64(len(b)))

    total := 0
    n, _ := file.Write(lb)
    total += n

    n, _ = file.Write(b)
    total += n

    return total, nil
}
```

> Note: Error handling is omitted for brevity, unless necessary. Please see source code for the complete implementation.

Similarly, we do this with our values, and end up with the following format. As a follow up, we can also add a checksum (CRC) before the key-value pair to help detect if the entry is corrupted.

```go
+-----+------------+-----+--------------+-------+
| CRC | Key Length | Key | Value Length | Value |
+-----+------------+-----+--------------+-------+
```

Now, returning to the `SSTManager` struct seen earlier, this component will be responsible for all our SSTable-related operations.

```go
// sst_manager.go
type SSTManager struct {
    ssTables  [][]SSTable
    ssCounter int
    // ...
}

type SSTable struct {
    ID       int
    FileSize int
    DataFile io.ReaderAt
}
```

Let's take a look at the code for adding a SSTable, which shouldn't be too much of a surprise. We traverse over keys in our memtable, persisting each key-value pair by flushing both to file. Once we are done, we then append our new SSTable to the first (and latest) layer of tables.

```go
// sst_manager.go
func (sm *SSTManager) Add(mt Memtable) error {
    curCounter := sm.ssCounter
    sm.ssCounter++

    dataPath := filepath.Join(sm.dir, fmt.Sprintf("lsm-%d.data", curCounter))
    dataFile, _ := os.OpenFile(dataPath, WR_FLAGS, 0644)
    defer dataFile.Close()

    mt.Traverse(func(k string, v []byte) {
        kn, _ := flushBytes(dataFile, []byte(k))
        vn, _ := flushBytes(dataFile, v)
    })

    // Re-open for read-only.
    dataFile, _ = os.Open(dataPath)

    sm.ssTables[0] = append(sm.ssTables[0], SSTable{
        ID:       curCounter,
        DataFile: dataFile,
    })

    return nil
}
```

### Deletes

We can implement deletes using writes. Instead of actually deleting anything, we'll instead insert a special type of key-value pair, known as a **tombstone**. The tombstone marks that an entry has been deleted, for our purposes, we'll use `nil`.

```go
// lsm_tree.go
func (lt *LSMTree) Delete(key string) {
    lt.Put(key, nil)
}
```

## Reads

Getting a kv-pair from our database is relatively simple, we just need to find the **latest** copy of an entry. We iterate over each memtable in reverse chronological order until we find the entry, or repeat the procedure with the SSTables.

```go
// lsm_tree.go
func (lt *LSMTree) Get(key string) ([]byte, error) {
    // Search tables in reverse chronological order.
    for i := len(lt.tables) - 1; i >= 0; i-- {
        v, found := lt.tables[i].Find(key)
        if found {
            return v, nil
        }
    }

    return lt.stm.Find(key)
}
```

```go
// sst_manager.go
func (sm *SSTManager) Find(key string) ([]byte, error) {
    for _, level := range sm.ssTables {
        for _, ss := range level {
            b, found, err := findInSSTable(ss, key)
            if err != nil {
                return nil, fmt.Errorf("unable to search in SSTables: %w", err)
            }
            if found {
                return b, nil
            }
        }
    }

    return nil, nil
}
```

Finding and determining whether a kv-pair exists in a SSTable involves iterating over **all** entries stored, which is sufficient for our current implementation. Each time we read a kv-pair, we must also increment the file offset.

```go
// sst_manager.go
func findInSSTable(ss SSTable, key string) ([]byte, bool, error) {
    maxOffset = ss.FileSize

    for offset < maxOffset {
        kvp, newOffset, err := readKeyValue(ss.DataFile, int64(offset))
        if err != nil {
            return nil, false, fmt.Errorf("unable to find in SSTable: %w", err)
        }
        offset = int(newOffset)

        if key == string(kvp.key) {
            return kvp.value, true, nil
        }
    }

    return nil, false, nil
}
```

To read a kv-pair from file, we perform a similar operation to `flushBytes` which we saw earlier. We first read in the length of the key or value, then using that, we read the key or value into memory.

```go
// data_file.go
func readBytes(file io.ReaderAt, offset int64) ([]byte, error) {
    lb := make([]byte, 8)
    file.ReadAt(lb, offset)

    length, _ := binary.Varint(lb)

    b := make([]byte, length)
    file.ReadAt(b, offset+8)

    return b, nil
}
```

```go
// data_file.go
func readKeyValue(file io.ReaderAt, offset int64) (keyValue, int64, error) {
    kb, _ := readBytes(file, offset)
    offset += 8 + int64(len(kb))

    vb, _:= readBytes(file, offset)
    offset += 8 + int64(len(vb))

    return keyValue{key: kb, value: vb}, offset, nil
}
```

And done! Now, we've got ourself a rudimentary, but working key-value store.

But, it's pretty clear that this is wildly inefficient once we have more than a few tables stored on disk, since we need to iterate over all entries in each one. Luckily, there's a few optimizations we can make to improve the lookup speed.

## Optimizations

### Sparse Indexes

One optimization we can make, is to reduce the number of entries we look through for each SSTable. Since entries in our table are already in sorted order, we can maintain a **sparse index** of entries (which denote an interval). And when we need to find an entry, we first perform a binary search to determine which interval to look in, and look *only* in that interval.

We will need to change how search is performed by setting the offsets to exactly the interval found.

```go
// sst_manager.go
func findInSSTable(ss SSTable, key string) ([]byte, bool, error) {
    offset, maxOffset := ss.Index.GetOffsets(key)
    if maxOffset < 0 {
        maxOffset = ss.FileSize
    }

    for offset < maxOffset {
        // ...
    }

    return nil, false, nil
}
```

We also need to create the sparse index when persisting memtables to disk. Here, the `sparseness` controls how large our intervals will be, and smaller values trade better performance for larger overhead.

```go
// sst_manager.go
func (sm *SSTManager) Add(mt Memtable) error {
    // ...

    si := NewSparseIndex()

    offset := 0
    iter := 0
    mt.Traverse(func(k string, v []byte) {
        kn, _ := flushBytes(dataFile, []byte(k))
        vn, _ := flushBytes(dataFile, v)

        if iter%sm.sparseness == 0 {
            si.Append(recordOffset{Key: k, Offset: offset})
        }

        offset += kn + vn
        iter++
    })

    // ...
}
```

Here I'm omitting the details of how the sparse index is implemented, but it is just a wrapper around a slice of `(key, offset)`, with a method that performs binary search to find the lower and upper bounds.

### Bloom Filters

Apart from reducing the number of records our key-value store needs to look at, we can also completely ignore some tables **if** we can determine if our key resides in the table. However, that would take at least as much space as there are keys, so clearly that would not be feasible.

But, we can do something almost as good. We can determine with *some probability* whether our key exists in a given set (and hence our table)! This is done using a **bloom filter**, a probabilistic data structure. I won't go into the details of implementing one or how it works, but feel free to check out [this](https://llimllib.github.io/bloomfilter-tutorial/) article I referenced.

The property that we want is that if the filter returns false, then we are *guaranteed* that our key does not exist in the table. But, if its true, then it *might* be in the table. 

Similarly to the previous optimization, we need to update `Add` and `findInSSTable`. First, we will just return `False` if the key does not exist in the bloom filter.

```go
// sst_manager.go
func findInSSTable(ss SSTable, key string) ([]byte, bool, error) {
    if ss.BloomFilter != nil && !ss.BloomFilter.In([]byte(key)) {
      return nil, false, nil
    }

    // ...

    return nil, false, nil
}
```

And let's not forget to add the key to the set while we persist the table!

```go
// sst_manager.go
func (sm *SSTManager) Add(mt Memtable) error {
    // ...

    si := NewSparseIndex()
    bf, _ := NewBloomFilter(mt.Nodes(), sm.errorPct)

    offset := 0
    iter := 0
    mt.Traverse(func(k string, v []byte) {
        kn, _ := flushBytes(dataFile, []byte(k))
        vn, _ := flushBytes(dataFile, v)
        bf.Add([]byte(k))

        if iter%sm.sparseness == 0 {
            si.Append(recordOffset{Key: k, Offset: offset})
        }

        offset += kn + vn
        iter++
    })

    // ...
}
```

Now it should be much better!

> For both optimizations, they add a certain amount of storage overhead. But this can be tuned by adjusting how sparse/dense we want our sparse index and the error rate of the bloom filter.

## Compaction

I'd like to first preface that compaction is a *very complicated* topic, and would require an entire post to explain all the various intricacies, and frankly one beyond my limited understanding. But, here is a very well written [paper](https://arxiv.org/pdf/2202.04522.pdf) that goes into detail on all the different aspects and considerations.

But first, why compaction? Well, recall from earlier when we implemented deletions using writes and tombstones. That creates a lot of "wasted" space in tables as more entries are deleted, so we would like to have some way to collect and "compact" older entries to free up space.

Another reason (which is cited in the paper), is that it is generally faster to perform a lookup on a larger table than over multiple tables.

For the scope of this post, we'll discuss the details of how multiple tables are compacted into one.

```go
// sst_manager.go
func (sm *SSTManager) compactTables(newID int, tables []SSTable) SSTable {
    kfh := make(KeyFileHeap, len(tables))
    level := tables[0].Meta.Level

    for i, t := range tables {
        kvp, offset, _ := readKeyValue(t.DataFile, 0)
        kfh[i] = KeyFile{
            Key:     string(kvp.key),
            Value:   kvp.value,
            FileIdx: i, // NOTE: this file does not represent the FileID.
            Offset:  int(offset),
        }
    }
    heap.Init(&kfh)

    dataPath := filepath.Join(sm.dir, fmt.Sprintf("lsm-%d.data", newID))
    dataFile, _ := os.OpenFile(dataPath, WR_FLAGS, 0644)

    si := NewSparseIndex()
    bf, _ := NewBloomFilter()

    offset := 0
    iter := 0
    prevKeyFile := KeyFile{}

    for len(kfh) > 0 {
        keyFile := heap.Pop(&kfh).(KeyFile)

        if keyFile.Key != prevKeyFile.Key && string(keyFile.Value) != "" {
            if iter%sparseness == 0 {
                si.Append(recordOffset{
                    Key:    string(prevKeyFile.Key),
                    Offset: offset,
                })
            }

            kl, _ := flushBytes(dataFile, []byte(prevKeyFile.Key))
            vl, _ := flushBytes(dataFile, prevKeyFile.Value)
            bf.Add([]byte(prevKeyFile.Key))

            offset += kl + vl
            iter++
        }

        prevKeyFile = keyFile

        if keyFile.Offset >= tables[keyFile.FileIdx].FileSize {
            continue
        }

        kvp, newOffset, _ := readKeyValue(
            tables[keyFile.FileIdx].DataFile,
            int64(keyFile.Offset),
        )

        heap.Push(&kfh, KeyFile{
            Key:     string(kvp.key),
            Value:   kvp.value,
            FileIdx: keyFile.FileIdx,
            Offset:  int(newOffset),
        })
    }

    // We need to remember to do the last one.
    kl, _ := flushBytes(dataFile, []byte(prevKeyFile.Key))
    vl, _ := flushBytes(dataFile, prevKeyFile.Value)
    bf.Add([]byte(prevKeyFile.Key))
    offset += kl + vl

    // ...
}
```

The above looks like a handful, but it is not all that different than what we did earlier in `Add`. Since each table is already sorted, we can use a min heap to advance over all tables at once while preserving order. Though not shown here, but the heap also uses chronological ordering to break ties so that earlier entries are processed first.

We keep track of the previous `keyFile` element and only flush it to the compacted table once its key is different than the current `keyFile` and if it is not a tombstone. The rest is the exact same as before, append to the sparse index, and add it to the bloom filter. Now, after each iteration, we add the next key-value entry from the same file back into the heap so we can repeat the process.

And at the end, we need to remember to flush the last key-value pair.

### Triggers

For our implementation, we will have the user manually invoke compaction. But this can also be done in more automated ways, such as when the number of tombstones exceed a certain threshold, or if tables are below a certain usage.

## Concurrency

So all the code up to this point have been presented with the assumption that there is a single caller. But what if we want to support concurrent operations? Apart from using `sync.RWMutex` in the necessary areas, there's a few small things to watch out for.

We want to be able to perform operations even while memtables are being flushed to disk, and while compactions are happening. Thankfully, because nonactive memtables are immutable and read-only, we only need to acquire the mutex before and after flushing, and not during.

```go
// lsm_tree.go
func (lt *LSMTree) flushPeriodically() {
    // ...
    
    lt.mu.Lock()
    var mts []Memtable
    n := len(lt.tables)
    if n > lt.maxMemTables {
        mts = lt.tables[:n-lt.maxMemTables]
    } else {
        lt.mu.Unlock()
        continue
    }
    lt.mu.Unlock()

    for _, mt := range mts {
      // Flush table.
    }

    lt.mu.Lock()
    lt.tables = lt.tables[n-lt.maxMemTables:]
    lt.mu.Unlock()

    // ...
}
```

For reads, we only acquire locks while reading memtables. This way, reads to SSTables can be blocked at the `SSTManager`, while not blocking subsequent reads and writes (to memtables).

```go
// lsm_tree.go
func (lt *LSMTree) Get(key string) ([]byte, error) {
    lt.mu.RLock()
    for i := len(lt.tables) - 1; i >= 0; i-- {
        v, found := lt.tables[i].Find(key)
        if found {
            lt.mu.RUnlock()
            return v, nil
        }
    }
    lt.mu.RUnlock()

    return lt.stm.Find(key)
}
```

## Remarks

Hopefully you enjoyed reading this as much as I did building it! I'm hoping to expand on this project in the near future by making a distributed kv-store, but that's still quite far away.

### Notes

- For the sake of clarity, I chose not to include any of the finer details on encoding and decoding files, sparses indexes or bloom filters, but they are included in the source.
- If you're interested in other kv-store architectures, be sure to check out [Bitcask](https://riak.com/assets/bitcask-intro.pdf). The entire paper is only 6 pages long, and very easy to implement.
