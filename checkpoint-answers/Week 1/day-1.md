# Correctness and Concurrency

## Why doesn’t the memtable provide a delete API?
A delete API is not necessary since is is not sematically any different than writing to a key. The tombstone that is created is also needed to ensure that the storage engine does not return an older value from before the deletion (since no key appears the same as not updated in the active memtable).
## Does it make sense for a memtable to store every write instead of only the latest version of a key? For example, suppose a user writes a -> 1, a -> 2, and a -> 3 to the same memtable.
If historical values are of interest, it would be necessary to store the additional writes. Assuming that there isn't any versioning and/or transactions going on it should not be necessary to store multiple values.
## Why do we need a combination of state and state_lock? Can we only use state.read() and state.write()?
While it might be possible to perform the coordination using only the locks already on state, it would require the locks to be held for a significantly longer amount of time. At best this structure would cause sporadic latency spikes when it needed to do work (like freezing the table or doing any IO).
## Construct the smallest example in which probing memtables in the wrong order returns a stale value. Then construct one in which it resurrects a deleted value. 
memtable a-> 1 imm_memtable_0 a-> 2, probing imm_memtable first would return 2. memtable b-> null imm_memtable_0 b -> 2, probing imm_memtable first would ressurect the value.
## After a memtable is frozen, could a thread that still holds an old LSM-state snapshot write to that now-immutable memtable? How does your solution prevent this?
It could if you do not make sure that there are no longer threads able to access that version. My solution ensures that there are no other accessors by requesting a write lock after acquiring the state_lock. Since there can only be 1 writer or multiple readers, acquiring the write-lock means that there can be no other users of that state.
## In several places, you might acquire a state read lock, release it, and then acquire a write lock. The two operations may occur in different functions that call one another. How does this differ from directly upgrading a read lock to a write lock? Is an upgrade necessary, and what does it cost?
It is no longer an atomic operation if the lock gets released. Other threads might have access to the contended information while the given thread waits to be able to get its write lock. An upgrade is not necessary, but it does require making sure that the state that the thread intended to act on is still requires the action.

# Performance and Design
## Could an LSM tree use other data structures for its memtable? What are the advantages and disadvantages of a skiplist?
Other structures could be used instead of a memtable. Memtables are efficent at highly concurrent writes/reads, not needing locking for either, but this lack of locks can allow race conditions if some sort of order of accesses is assumed.
## Is the memtable’s memory layout efficient? Does it have good data locality? Consider how Bytes is implemented and stored in the skiplist. How could you optimize the memtable’s layout?
The memtable does not necessarily have the best data locality, since the keys and values will live in unrelated places in the heap due to the keys not being owned by the insertion function. One way to improve the locality could be to investigate using a shared buffer as the source for the accessed bytes structs to point into, allowing the bytes data pointers to point into closer locations.
## Documentation check: Read parking_lot’s RwLock fairness section. What might happen to readers waiting to acquire the lock when a writer is already waiting for the current readers to release it? How does eventual fairness differ from strict first-in, first-out service?
The readers may be made to wait if the timeout has been reached and the lock is being acquired for the first thread in line. Eventual fairness will still allow barging to a degree, but will force FIFO-like assignment around every ~0.5ms
## Use the Week 1 end-of-week self-check to calibrate the core invariants. The remaining design questions may have several defensible answers; state your workload and assumptions before comparing tradeoffs. You can also discuss them in the Discord community.

# Bonus Tasks
## More Memtable Formats. Implement other memtable formats, such as B-tree, vector, or adaptive radix tree (ART) memtables.
