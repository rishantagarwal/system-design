# The Google File System
The Google File System (GFS) is a scalable, distributed file system.
## Overview
* GFS is designed for files that are 100 MB or larger. Multi-GB files are common. GFS is not optimized for small files.
* Writes are typically large, sequential writes that are appended to files. Once written, files are seldom modified again.
* Most client applications prioritize processing data in bulk at a high rate. Not many have low latency requirements for an individual read/write.
## Design
* Each GFS cluster has a single master and multiple chunk servers (3 replicas at default). 
### Master
* In terms of data, the master only maintains the file system metadata.
* The master sends HeartBeat messages to each chunkserver to provide instructions and collect state.
* As metadata is minimal compared to chunks, the master is able to keep all metadata in memory. This allows master operations to be fast.
### Chunkservers
* Chunkservers do not cache file data. Files are too large to be cached.
### Clients
* Clients initially interact with the master for metadata. After that, clients communicate directly with the chunkservers
* Clients only cache metadata. Clients stream huge files or have working sets too large to be cached.
### Chunk Size
* Files are divided into fixed-size 64 GB chunks. Each chunk is identified by a globally-unique 64 bit chunk handle.
* Large chunk size reduces client interactions with the master. Reads and writes on the same chunk only require one initial request to the master for chunk location info.
### Metadata
* The metadata consists of file and chunk namespaces, access control information, file-to-chunk mappings, and chunk locations (on all replicas).
* All metadata is available in master's memory.
* Namespaces and file-to-chunk mappings are persisted through log mutations on an operation log stored on master's local disk. The operation log is also replicated on remote machines. During master startup, it replays the operation log to recover its file system state.
#### Chunk Locations
* Chunk locations are not persistently stored on master's disk. Chunkservers provide chunk location information to the master. When a master starts up, it polls the chunkservers for this info. When a chunkserver joins a cluster, it provides this info to the master.
* The master stays up-to-date on chunk locations through periodic HeartBeat messages.
#### Operation Log
* When the operation log becomes large, the master will checkpoint its state.
* Master startup time is minimized by keeping the log small and replaying from the last checkpoint.
* The checkpoint is a compact B-tree. Older checkpoints are deleted after the creation of a new checkpoint.
## Consistency Model
* GFS has a relaxed consistency model.
* Mutations to a chunk are applied in the same order across all replicas.
* Chunk version numbers are used to detect replicas that have become stale. Stale replicas are garbage collected at the earliest opportunity.
* Whenever a master grants a new lease on a chunk, it increases the chunk version number and informs the replicas. Chunk version numbers are applied and persisted on master and the replicas prior to client notification of the lease.
* Leases include the chunk version number. Clients and chunkservers can verify the version number prior to performing an operation.
* The master detects failed chunkservers by detecting data corruption through checksumming.
* GFS accomodate the relaxed consistency model by relying on appends versus overwrites, checkpointing, and writing self-validating, self-identifying records.
* Each record prepared by the writer contains checksums so the record's validity can be verified.
## Leases and Mutation Order
These are the steps in which mutations are applied.
1. The client requests for the primary chunkserver that hold a lease to a chunk as well as the replicas.
1. If a chunkserver doesn't have the lease, the master grants a lease to one of the replicas. This replica becomes the primary.
1. The client caches this lease for future mutations until the lease times out.
1. **Data Flow.** The client pushes the data to all the replicas. Each chunkserver stores the data in an internal LRU buffer cache.
1. **Control Flow.** Once all the replicas acknowledge receiving the data, the client sends a write request to the primary.
1. The primary assigns a consecutive serial number to the mutation (serialization). It applies the mutation in its own local state in serial number order.
1. The primary forwards the write request and serial number to all the secondary replicas. Each replica applies the mutation in the same serial number order as the primary.
1. The secondary replicas reply to the primary indicating the write operation was successful.
1. The primary replies to the client. Any errors are reported accordingly. If the write failed, the client will retry the operation from step 1.