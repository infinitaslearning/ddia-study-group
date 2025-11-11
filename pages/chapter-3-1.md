# Storage and Retrieval

On the most basic level a database does only two things:
- stores some data
- when asked, gives that data back

---

# Who cares about database engines?

> Why should I care about databases engines and how they are made?

Because as developer you should know which database is more appropriate to your application requirements.

Every engine has his own strenghts and weaknesses, so it's important to make the right choice.

For example some engines are optimized for analytics, some for transactional workloads, so you need to have a rough idea of what your database is doing under the hood.

---

# Let's start from scratch

Let's build our first database

```bash
#!/bin/bash

db_set () {
		echo "$1,$2" >> database
}

db_get () {
	grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

It may look trivial, but it works!

---

# Our database in action


<img src="../assets/chapter03/c3-01.jpg" alt="database content" class="w-150 center m-auto"/>

---

# It's great because

- it's simple 

- good WRITE performances

append to file is fast



# It's also terrible because

- bad READ performance 

- ...and it gets worse the more data is stored [O(n)]

- space used only keeps growing

- no concurrency control

- no error handling/partially written records/...

---

# HOW CAN WE ACHIEVE FASTER READS?
<br/>
<br/>
<img src="../assets/chapter03/c3-02.jpg" alt="database content" class="w-150 center m-auto"/>
<br/>

## Why the read operation is slow?

Because in order to find the key we're looking for, we need to scroll through the whole data.

## What can we do to fix that?

Keep some additional metadata on the side which acts as signpost to quickly tlocate the data we want

---

# Hash Indexes

An index is a derived data structure with some metadata that helps navigate through the database contents.

<br/>
<img src="../assets/chapter03/c3-03.jpg" alt="database content" class="w-150 center m-auto"/>
<br/>

We're adding a simple in-memory hashmap
mapping each key to his specific byte position inside the file.

So when we need to access to the value of a key:

- instead of looping through each character of the store
- we can jump straight to the correct index, because the index tell us where it is.

---

# Hash Indexes

- An index greatly improves the read performances
- The cost is moved to the write operation, any time some data is written, you must update the index as well.

> This tradeoff is the reason it is not enabled by default everywhere. The developer must decide when and where is appropriate to enable it.

- Indexes can be added or removed without affecting the main data.

- If you need to search through data on multiple different keys, then you might need multiple indexes

- This approach shines when you have a large amount of writes per key

> (AND you have enough available memory to store all the keys)


This might look simplistic, but it's a real viable approach (see Bitcask - the default storage engine in Riak)

---

# HOW CAN WE STOP WASTING DISK SPACE

On the first implementation of our db:
- when we udpate a record, we add a new line on the same key
- when we delete a record, we add a new line with a tombstone special character

With this append-only storage even delete operations add new data to the disk

It works but our disk space used only keeps growing with time; with a big enough dataset this is not sustainable.

---

# COMPACTION PROCEDURE

We cycle through the whole storage file and remove "useless" rows (the one that get overwritten by new values on the same key).

<br/>
<img src="../assets/chapter03/c3-04.jpg" alt="database content" class="w-130 center m-auto"/>
<br/>

To apply this to our store we need to:

- break the log into segments of a certain size
- when we reach the size limit we move into a new block
- then we apply compaction between the blocks (can be done in the background)
- we now merge the output into new segment files
- we can now delete the old segments (and free some disk space)

---

# COMPACTION PROCEDURE

- Each segment must own his dedicated index table
- Indexes must be re-calculated based on the new compacted file version

<br/>
<img src="../assets/chapter03/c3-05.jpg" alt="database content" class="w-100 center m-auto"/>
<br/>

## QUESTION TIME:

why should I split my store in segments and not just keep it unified instead?

---

# Other boring details in real-life scenarios

FILE FORMAT:

> CSV is not be the best format for a log.

> It's faster and simpler to use a binary format that defines a string length before the actual string (in order to avoid escaping)


CRASH RECOVERY:

> On restart you should re-calculate all the hashes, but this can be long and painful on large dataset.

> A simple copy of the current index map on disk can speed things up for good.


PARTIALLY WRITTEN RECORDS:

> If a crash happens during a write, we need a way to detect broken logs and ignore them.


CONCURRENCY CONTROL:

> Write are simple append in a strict sequential order, --> requires a single dedicated thread.

> Read can run in parallel threads

---

# APPEND-ONLY STORAGE NOTES:

Append-only store might seem wasteful at first glance but are actually pretty good for various reasons:

- Append and merge are sequential write operations
so they are very fast on magnetic, disk spinning and sometimes even on SSD hard drives

- Concurrency and crash recovery are easy to manage with this write model

- Segment merging automatically solves the problem of data fragmentation on disk

But they are also unfit on some scenarios:

- *The hash table must fit in your memory*, so a large amount of keys might be a problemino
(you should move the hashmap to disk which is slow and painful, has terrible random access performances, no hash key collision mechanism, yadda yadda, ...)

- *Range queries are not efficient*
so if you need to fetch all HELLO_KITTY keys from HELLO_KITTY_0001 to HELLO_KITTY_9999
each key must be obtained from the hashmap
and your HELLO_KITTY consumption is slowed down by a good margin

---

# SSTables and LSM-Trees

AKA Sorted String Table and Log-Structured Merge-Tree

starting from the previous implementation we will sort the rows by key.

1. We no longer need an index of all keys in memory.
> We still need an index file, but it include only some of the keys, not all.
Just jump to the closest index and start searching from there.

2. Merging segments together is simple and efficient
> even if the files are bigger than available memory
(just read the two segments together and take the lowest value every time
if the same key appears twice, take it from the most recent segment)

3. It is possible to group records into a block and compress them before writing them into disk.
> less disk space used also means less I/O bandwidth usage, too

---

# B-Trees

---

# B-Trees vs LSM-Trees

---

# values in the index

heap file

---

# Multi-column indexes

---

# Full-text search and fuzzy indexes

---

# In memory indexes
