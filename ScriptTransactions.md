---
RFC: (PR number)
Status: (Change to Proposed when it's ready for review)
---

# EVAL/FCALL Transactions

## Abstract


## Motivation

Currently the scripts executed by the `EVAL` and `FCALL` commands run atomically because the execution is single-threaded but if some error occurs, the changes made by the script are not rolled back, unless the script implements logic to rollback the changes. 
But even if the script has logic for rolling back the changes, it is not enough in the case where the script is killed by an external command like `SCRIPT KILL` or `FUNCTION KILL` because the script cannot detect that it has been killed.

The goal of this new feature is to add the support for automatic rollback of changes made to the Valkey database that resulted from the execution of scripts.

This is also a long time request by some companies like the one reported in https://github.com/redis/redis/issues/10576.


## Design considerations

We can implement an undo-log approach where the new values are added changed in place while still keeping the old values, and uppon commit, they become the definite values on the data structure. And in case of a rollback, the old values are restored.

A naive implementation of this approach will introduce a big overhead, or stall, when rolling back the values because if the number of values changed is very high, it will take time to rollback each value.
But we can use an implementation based on lock-free algorithms where we can "rollback" the values in a single step, using pointer indirection.

In practice, we only need to maintain two versions of each value. The current value, and the old value (in case that the value was updated).
Somekind of garbage collection mechanism must exist to cleanup the versions corresponding to old values in the case of a transaction commit, and new values in the case of a transaction rollback.
This GC process may execute in the background, or in the foreground but incrementaly.

We need to implement this versioning scheme for database keys. This means that we may duplicate the memory size of each key value that is changed by a script.
Since there might exist values that are very big memory-wise, the data types of these values may optionaly implement transactional semantics by also using the same versioning scheme.
For instance, if the list data type implements the transactional semantics, instead of duplicating the database key value that stores the list, only the new value added to a list needs to be versioned.

The module API for implementing new data types is also extended to allow the data type implementation declare that it implements the transactional semantics.


## Specification




### Commands (Optional)

If any new commands are introduced:

1. Command name
   - **Request**
   - **Response**

### Authentication and Authorization (Optional)

If there are any changes around introducing new ACL command/categories for user access control.

### Append-only file (Optional)

If there are any changes around the persistence mechanism of every write operation.

### RDB (Optional)

If there are any changes in snapshotting mechanisms like new data type, version, etc.

### Configuration (Optional)

If there are any configuration changes introduced to enable/disable/modify the behavior of the feature.

### Keyspace notifications (Optional)

If there are any events to be introduced or modified to observe activity around the dataset.

### Cluster mode (Optional)

If there is any special handling for this feature (e.g., client redirection, Sharded PubSub, etc) in cluster mode or if there are any new cluster bus extensions or messages introduced, list out the changes.

### Module API (Optional)

If any new module APIs are needed to implement or support this feature.

### Replication (Optional)

If there are any changes required in the replication mechanism between a primary and replica.

### Networking (Optional)

If there are any changes introduced in the RESP protocol (RESP), client behavior, new server-client interaction mechanism (TCP, RDMA), etc.

### Dependencies (Optional)

If there are any new dependency libraries required to support the feature. Existing dependencies are jemalloc, lua, etc. If the library needs to be vendored into the project, please add supporting reason for it.

### Benchmarking (Optional)

If there are any benchmarks performed and preliminary results (add the hardware/software setup) are available to share or a set of scenarios identified to measure the feature's performance. 

### Testing (Optional)

If there are any test scenarios planned to ensure the feature's stability and validate its behavior.

### Observability (Optional)

If there are any new metrics/stats to be introduced to observe behavior or measure the performance of the feature.

### Debug mechanism (Optional)

If there is any debug mechanism introduced to support admin/operators for maintaining the feature.

## Appendix (Optional)

Links to related material such as issues, pull requests, papers, or other references.
