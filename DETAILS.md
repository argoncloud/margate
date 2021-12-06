## Node services

### Storing data
Raft has to run for the quorums, so the system needs an append-only log with the data.
mmap'd file, like:
```
timestamp, event_type
           <key_set>,  key, value (can be large!)
           <key_del>,  key
           <snap_set>, snapshot_name, snapshot_id
           <snap_del>, snapshot_name
           <borrow>, (key_range, key_range, ...)
           <return>, (key_range, key_range, ...)
```

### Finding data
LMDB (which is a tree sorted lexicographically on keys) keeps:
```
key -> (resource utilization fields
        snap_id0 version_id0 -> log_entry0
        snap_id0 version_id1 -> log_entry1
        snap_id1 version_id2 -> log_entry2
        snap_id1 version_id3 -> 00000/deleted)
```
ShardSes keep maps of snapshots IDs to translate back and forth from snapshot ids.

### Transactions
While the transaction is ongoing, we bring ownership for all keys in the transaction to one single ShardS.
All writes end up in the same place, and then we flush them to the original ShardS that own that range
when possible, returning ownership as we flush.

Process:
* Client picks a transaction leader (which is also a Shard leader)
  * presumably, the Shard that expects the most data written in the transaction
* Client leases all keys involved (read and write) to a single Shard leader by talking to all other Shard leaders
  * on tleader, `TRANSACTION_CREATE(transaction_id, [tx_keys]) -> status`
  * on bleaders, `LEASE_TAKE(transaction_id, leader_host, [local_keys]) -> status`
* Client does reads directly from Shard leaders
* Client does all writes to transaction leader
* Client marks the transaction complete (or reverted)
  * `TRANSACTION_COMMIT(transaction_id)`
  * `TRANSACTION_ROLLBACK(transaction_id)`
* transaction leader flushes all writes to corresponding Shard leaders at its leisure
  * each flush that succeeds releases the leases
    * `LEASE_RETURN_PLAIN(transaction_id, [keys]) -> status`
    * `LEASE_RETURN_WRITE(transaction_id, key, value) -> status`
* shard leaders periodically check if leases are still active (if transaction is still in progress)
  * this handles client failures
  * `LEASE_CHECK(transaction_id) -> still_leased, lease_unknown`
    * `lease_unknown` causes the leader to release the lease

### Snapshots
Send the snapshot request to all ShardSes involved. The Shard leaders increase the snapshot ID atomically
before or after other operations that are ongoing.

### Compactions
The system is already built around a reconstructible log, so compactions can be done by going over the log,
skipping entries that are not applicable anymore, and applying the events that are relevant onto a new set
of data structures, as if they were new. Finally, delete the old data structures entirely.


## Directory Services

### Coordinate Consistent Hash changes with clients
* Serve updated Consistent Hashes to Clients.
* Coordinate timings of updates; by updating the Consistent Hash on a predictable schedule, reduce
  chances of stale hits by Clients to ShardSes.

### Bundling Nodes to mint Shard quorums
* Run the optimization/solver to find `n` machines to bundle together.
* Consider placement groups to minimize similarity between Shards (e.g. two Shards with same Nodes would go down at the same time).

### Track resource usage to trigger rebalances
* Periodically review the load reports from Nodes.
* Identify Nodes that are over average, and find Shards that can be spread around.

### Replace troubled Nodes
* Respond to ShardS requests to replace nodes that became unresponsive.

### Service machine addition/removal from operations
* Trigger proactive moveouts.
