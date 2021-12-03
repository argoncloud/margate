# Margate
**_A strongly-consistent key/value and object store._**

## Client API
A key object can be a:
* `literal string`
* `child_min(literal string)`
* `child_max(literal string)`
* `child_next_b64u(literal string) (NOTE: this key never exists at the time of calling)`
* `lower_bound(literal string)`
* `higher_bound(literal string)`

Entry:
* `entry_set(key, value, create_only?, modify_only?, at_version?, key?, old_value?) -> version, key?, old_value?`
* `entry_get(key, version=LATEST, key?) -> version, key?, value`
* `entry_delete(key, at_version?, key?, old_value?) -> success, key?, old_value?`
* `entry_size(key, version=LATEST, key?) -> version, key?, size`
* `entry_cmpxchg(key, old_value, value, key?, old_value?) -> success, key?, old_value?`

Transactions:
* `txn_begin(ranges)`
* `txn_rollback()`
* `txn_commit()`

Keys:
* `range_list(key_low, key_high, values?) -> keys, values?`
* `range_count(key_low, key_high) -> count`
* `key_get(key) -> key (NOTE: use with non-literal keys)`

Snapshots:
* `snap_add(label, key_low, key_high)`
* `snap_del(label) -> success`

### Examples
```
// move()
txn_begin(["/inventory/a", "/inventory/b"])
if !(v = entry_delete("/inventory/a", old_value=true)) txn_rollback()
if !entry_set("/inventory/b", v, insert_only=true) txn_rollback()
txn_commit()
```
```
// increment()
txn_begin(["/inventory/a"])
if !(v = entry_get("/inventory/a")) txn_rollback()
if !entry_set("/inventory/a", v+1) txn_rollback()
txn_commit()
```
```
// enqueue()
entry_set(child_next_b64u("/inventory/queue/"), "some data")
```
```
// dequeue()
v = entry_delete(child_min("/inventory/queue/"), old_key=true, old_value=true)
```
```
// useless trick... move() last under tree to next
old_key = child_max("/inventory/tree/")
new_key = child_next_b64u("/inventory/tree/")
txn_begin([old_key, new_key])
if !(v = entry_delete(old_key, old_value=true)) txn_rollback()
if !entry_set(new_key, v, insert_only=true) txn_rollback()
txn_commit()
```

## Glossary

| Term | |
| --- | --- |
| Consistent Hash | Data structure with the mapping of Key ranges -> Nodes. |
| Deployment | A set of machines that operate together to span a whole key universe. |
| Key Range | A consecutive, uninterrupted range in the Key Space... doesn't need to start or end at an existing key. |
| Key Space | The set of all keys possible for a Deployment. This is (theoretically) unbounded. Keys are sorted lexicographically within it. |
| Node | A NIC within one machine. |
| Quorum | A set of processes that participate in a consensus quorum. |
| Shard | A Quorum of Shard Instances that own a Key Range. |
| Shard Instance | Logic on a Node that tracks a replica of a Shard. |
| Transaction Leader | A Node that owns the completion of a transaction. |

## Components

### Directory Service
A Quorum of instances that own the Consistent Hash.
* One per Deployment.
* Tracks the list of machines that participate in the Deployment.
* Tracks resources utilization on each machine.
* Enforces that each machine has resource utilization close to the average, and otherwise Rebalances.
* Responds to Clients that want to cache the Consistent Hash.


### Node
* Handles Client queries.
* Compacts its data to reclaim space as needed.
* Acts as Transaction Leader when picked by a Client.
* Reports its load to Directory Service periodically.
* Moves data from its shards to other nodes when directed by the Directory Service to rebalance.

### Shard
* Identified by an ever-growing id.
* Contains a list of Nodes that participate, and the Key Range it manages.

### Shard Instance
* Runs Raft together with the other Shard Instances.
* As a Leader:
  * reports becoming leader to the Directory Service.
  * manages replication, and requests replacements of Shard Instances that became unavailable.
* As a Follower:
  * checks liveness of the leader, and potentially proposes itself as a replacement.
  * applies changes communicated by the leader.

### Potential future additions
* Directory Router, sitting between Clients and Shard Instances. Offloads large Consistent Hashes from Clients, and routes appropriately.

Copyright © Claudio A Andreoni, 2021.
