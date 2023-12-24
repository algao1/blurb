---
title: "Implementing a Key-Value Store From Scratch"
date: 2023-08-23
author: "agao"
ShowToc: true
---

A few months ago I was inspired by [this post](https://artem.krylysov.com/blog/2023/04/19/how-rocksdb-works/) on how RocksDB works, and decided that it would be a fun exercise to implement a database myself. This post is more of a documentation of how I built my own key-value store, but I still hope to give an overview of how they work, and provide enough details for someone to implement one themselves. 

The database and code snippets are written in Go, and the source code can be found [here](https://github.com/algao1/crumbs/tree/master/dbs/lsm).

## Log Structured Merge Tree

For our implementation, we'll be using a log structured merge tree (or LSM tree). It is a data structure that is comprised of two other data structures, **memtables** (an in-memory data structure that stores key-value pairs) which exists in memory, and **SSTables** (sorted strings tables, which we'll explore later) which exists strictly on disk. 

The memtables are kept in the first layer of the tree, and sorted in chronological order. Similarly, the SSTables are divided into different layers from newest to oldest.

![LSM Tree](/blurb/img/lsm_tree.png)

This setup allows our writes to be sequential since we can just write to the latest table, and append new ones as the old ones fill up (which makes this fast!).

```go
// lsm_tree.go
type LSMTree struct {
    tables []Memtable
    stm    *SSTManager
    // ...
}
```


## Writes

Like mentioned above, writes (or inserts) to the database are appended directly to the newest **memtable**. Once that memtable reaches a certain threshold, it is rotated out for a new one. Every memtable except the latest one is immutable and read-only (we'll see why shortly).

A memtable is just an interface, and any structure that implements finding/inserting a key-value pair ([AA tree](https://user.it.uu.se/~arnea/ps/simp.pdf), red-black tree, skiplist, etc.) and listing all kv-pairs in order, is sufficient.

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

If the number of memtables exceeds a given threshold, then we want to persist that to disk as a SSTable. This can be done in several ways, but for simplicity, I've opted to have a background process that periodically flushes extra tables to disk (not shown above, but you can check it out [here](https://github.com/algao1/crumbs/blob/7b623c503e7ea02ac3c43887fbeb28ab54b56a21/dbs/lsm/lsm_tree.go#L137-L168)).

### Persisting Memtables

We persist our memtable into a **SSTable**, which is just a subset of key-value pairs (those from the memtable) in a **sorted order** (hence the name, *sorted strings table*). Since it is stored on disk, we need to encode it in a format that is compact, and easy to retrieve.

For our purposes, we want to support variable-lengthed keys, so we'll first encode the length of the key (which is fixed to 8 bytes), and then the actual key.

```go
// data_file.go
func flushBytes(file *os.File, b []byte) (int, error) {
    lb := make([]byte, 8)
    binary.PutVarint(lb, int64(len(b)))

    // Write the length of the key/value.
    bytesWritten := 0
    n, _ := file.Write(lb)
    bytesWritten += n

    // Write the key/value.
    n, _ = file.Write(b)
    bytesWritten += n

    return bytesWritten, nil
}
```

> Note: Error handling is omitted for brevity, unless necessary. Please see source for the complete implementation.

Similarly, we do this with our values, and we end up with the following format for our key-value entries.

```go
+----------------+------------+-----+--------------+-------+
| CRC (Optional) | Key Length | Key | Value Length | Value |
+----------------+------------+-----+--------------+-------+
```

> (Optional): As a follow up, we can also add a checksum (CRC) before the key-value pair to help detect if the entry is corrupted.

Now, returning to the `SSTManager` struct seen way earlier, this component is responsible for handling all our SSTable-related operations.

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

Let's take a look at the code for adding a SSTable, which shouldn't be too much of a surprise. We first traverse over keys in our memtable, persisting each key-value pair by flushing both to file. Once we are done, we append our new SSTable to the first (and newest) layer of tables.

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

We can implement deletes using writes. Instead of actually deleting anything, we'll instead insert a special type of key-value pair, known as a **tombstone**. A tombstone marks that an entry has been deleted, and for our purposes we'll use the `nil` value.

```go
// lsm_tree.go
func (lt *LSMTree) Delete(key string) {
    lt.Put(key, nil)
}
```

We do this because **1.** it is easy to implement, and **2.** because removing a pair from our memtable usually requires rebalancing or some other overhead (depending on the implementation).

## Reads

Getting a kv-pair from our database is relatively simple, we just need to find the **latest** copy of an entry (remember that we never delete/update old copies once persisted!). We iterate over each memtable in reverse chronological order until we find the entry. Otherwise, we repeat the same procedure with the SSTables.

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
            b, found, _ := findInSSTable(ss, key)
            if found {
                return b, nil
            }
        }
    }

    return nil, nil
}
```

Finding and determining whether a kv-pair exists in a SSTable involves iterating over **all** entries stored, which will be sufficient for our current implementation. We do this by loading the entire file into a buffer, and then reading from it one pair at a time.

```go
// sst_manager.go
func findInSSTable(ss SSTable, key string) ([]byte, bool, error) {
	chunk, err := readChunk(ss.DataFile, 0, ss.FileSize)
	if err != nil {
		return nil, false, fmt.Errorf("unable to read chunk: %w", err)
	}
	buf := bytes.NewBuffer(chunk)

    for buf.Len() > 0 {
		kvp, _, err := readKeyValue(buf)
		if err != nil {
			return nil, false, fmt.Errorf("unable to find in SSTable: %w", err)
		}
		if key == string(kvp.key) {
			return kvp.value, true, nil
		}
    }

    return nil, false, nil
}
```

To read a kv-pair from the buffer, we need to read both using `readElement`, which is similar to the `flushBytes` we saw earlier. We first read in the length of the element (key or value), then using the length we read the element into memory.

```go
// data_file.go
func readElement(reader io.Reader) ([]byte, int, error) {
    lengthBytes := make([]byte, 8)
    reader.Read(lengthBytes)
    
    length, _ := binary.Varint(lb)

    b := make([]byte, length)
    reader.Read(b)

    // Return the element, number of bytes read, and error.
    return b, int(8 + length), nil
}
```

```go
// data_file.go
func readKeyValue(file io.ReaderAt) (keyValue, int64, error) {
    kb, kSize, _ := readElement(reader)
    offset += 8 + int64(len(kb))

    vb, vSize, _ := readElement(reader)
    offset += 8 + int64(len(vb))

    return keyValue{key: kb, value: vb}, kSize + vSize, nil
}
```

And done! Now, we've got ourself a rudimentary, but working key-value store.

However, it's pretty clear that this is wildly inefficient once we have more than a few tables stored on disk, since we need to iterate over all entries in each one. Luckily, there's a few optimizations that we can make to improve the lookup speed.

## Optimizations

### Sparse Indexes

One optimization we can make, is to reduce the number of entries we have to look through for each SSTable. Since entries in our table are already in sorted order, we can maintain a **sparse index** of entries (each denoting an interval). And when we need to find an entry, we can perform a binary search to determine which interval (chunk) to look in, and look **only** in that interval.

We will need to change how search is performed by setting the offsets to exactly the interval found.

```go
// sst_manager.go
func findInSSTable(ss SSTable, key string) ([]byte, bool, error) {
    offset, maxOffset := ss.Index.GetOffsets(key)
    if maxOffset < 0 {
        maxOffset = ss.FileSize
    }

    chunk, err := readChunk(ss.DataFile, offset, maxOffset-offset)
	if err != nil {
		return nil, false, fmt.Errorf("unable to read chunk: %w", err)
	}
	buf := bytes.NewBuffer(chunk)

    for buf.Len() > 0 {
        // ...
    }

    return nil, false, nil
}
```

We also need to create the sparse index when persisting memtables to disk. Here, the `sparseness` parameter controls how large our intervals will be, and smaller values trade better performance for larger overhead (up until a certain point, then it actually gets slower due to making syscalls to read chunks).

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

        // Here!!
        if iter%sm.sparseness == 0 {
            si.Append(recordOffset{Key: k, Offset: offset})
        }

        offset += kn + vn
        iter++
    })

    // ...
}
```

Here I'm omitting the details of how the sparse index is implemented, but it is essentially a wrapper around a slice of `(key, offset)`, with a method that performs binary search to find the appropriate lower and upper bounds.

### Bloom Filters

Apart from reducing the number of records our key-value store needs to look at, we can also completely ignore some tables **if** we can determine whether our key resides in the table. However, that would usually take at least as much space as there are keys, so clearly that would not be feasible.

But, we can do something almost as good. We can determine with *some probability* whether our key exists in a given set (and hence our table)! This is done using a **bloom filter**, a probabilistic data structure. I won't go into the details of implementing one or how it works, but feel free to check out [this](https://llimllib.github.io/bloomfilter-tutorial/) article I used for my implementation.

The property that we want is that if the filter returns false, then we are *guaranteed* that our key does not exist in the table. But, if its true, then it *might* exist in the table. 

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

        // Here!!
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

For the scope of this post, we'll discuss the details of how multiple tables are compacted into one. Below is the core logic for compacting tables:

```go
// sst_manager.go
func (sm *SSTManager) compactTables(newID int, tables []SSTable) SSTable {
    kfh := make(KeyFileHeap, len(tables))
    level := tables[0].Meta.Level
    totalItems := 0

    // Initialize the start of each table into heap.
    for i, t := range tables {
        kvp, offset, _ := readKeyValue(t.DataFile)
        kfh[i] = KeyFile{
            Key:     string(kvp.key),
            Value:   kvp.value,
            FileIdx: i, // NOTE: this file does not represent the FileID.
            Offset:  int(offset),
        }
        totalItems += t.Meta.Items
    }
    heap.Init(&kfh)

    dataPath := filepath.Join(sm.dir, fmt.Sprintf("lsm-%d.data", newID))
    dataFile, _ := os.OpenFile(dataPath, WR_FLAGS, 0644)

    si := NewSparseIndex()
    bf, _ := NewBloomFilter()

    offset := 0
    iter := 0
    prevKeyFile := KeyFile{}

    // While our heap is not empty, we have entries to iterate.
    // For each iteration, if it is different than the previous
    // entry and is not a tombstone, flush the old entry.
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

        kvp, size, _ := readKeyValue(tables[keyFile.FileIdx].DataFile)

        heap.Push(&kfh, KeyFile{
            Key:     string(kvp.key),
            Value:   kvp.value,
            FileIdx: keyFile.FileIdx,
            ffset:   keyFile.Offset + size,
        })
    }

    // We need to remember to flush the last one.
    kl, _ := flushBytes(dataFile, []byte(prevKeyFile.Key))
    vl, _ := flushBytes(dataFile, prevKeyFile.Value)
    bf.Add([]byte(prevKeyFile.Key))
    offset += kl + vl

    // ...
}
```

It looks like a handful, but it is not all that different than what we did earlier in `Add`. Since each table is already sorted, we can use a *minheap* to advance over all tables at once while preserving order. Though not shown here, but the heap also uses chronological ordering to break ties so that earlier entries are processed first.

We keep track of the previous `keyFile` element and only flush it to the compacted table once its key is different than the current `keyFile` and if it is not a tombstone. This prevents us from adding duplicate and deleted entries.

The rest is the exact same as before, we append to the sparse index, and add it to the bloom filter. Now, after each iteration, we add the next key-value entry from the same file back into the heap so we can repeat the process. And at the end, we need to remember to flush the last kv-pair.

### Triggers

For our implementation, we will have the user manually invoke compaction. But this can also be done in more automated ways, such as when the number of tombstones exceed a certain threshold, if a table is below a certain usage, or if the number of tables in a level exceeds a threshold. 

## Remarks

Hopefully you enjoyed reading this as much as I did building it! I'm hoping to expand on this project in the near future by making a distributed kv-store, but that's still quite far away.

One thing I've omitted from this blog is the [write ahead log](https://en.wikipedia.org/wiki/Write-ahead_logging) (WAL) which is used for crash recovery. I felt like it somewhat detracted from the main ideas of how a database worked, and I didn't have quite as much time to implement it.

- For the sake of clarity, I also did not include any of the finer details on encoding and decoding files, sparses indexes or bloom filters, but they are included in the source.
- If you're interested in other kv-store architectures, be sure to check out [Bitcask](https://riak.com/assets/bitcask-intro.pdf). The entire paper is only 6 pages long, and very easy to implement.
