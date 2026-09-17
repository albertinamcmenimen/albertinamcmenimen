## Unique Howell

Computer Science · Database Internals & Storage Engines

### Professional Focus

I design and implement storage engines whose invariants remain explicit under disk failures, process restarts, and concurrent access. My work spans LSM-tree indexing, page-oriented block storage, recovery, and the operational behavior of engines under sustained load. I optimize for crash consistency, bounded write amplification, predictable tail latency, and recovery that remains bounded by replayable state.

### Flagship Projects & Architecture

#### MerkleDB

A single-node key-value store that combines an in-memory LSM-tree index with a replicated local log and a CRC-32C Merkle index for bounded-memory repair.

**Architecture:** The engine uses a 128 KiB segment layout with fixed 4 KiB pages, sequence numbers, checksums, and a two-phase commit record for each committed key range. A single coordinator thread owns the index and applies bounded batches to a segmented append log; worker threads perform compaction and checkpoint I/O under producer-consumer limits. The on-disk format stores indexed tuples in page-aligned records and replays committed segments during recovery. Clients use a framed, length-delimited TCP protocol with a 64 KiB maximum payload, sequence numbers, checksums, and a compact binary serialization. Each segment is 64 MiB, and the index keeps only the current 128 MiB window plus a fixed-size Merkle frontier.

**Trade-offs:** I chose append-only segment writes over frequent random page updates to keep recovery replayable, and paid for higher storage overhead until compaction reclaimed unused space. I chose an in-memory index over a disk-resident B+ tree to keep lookup latency predictable, and paid for memory use that scales with the active key range. I chose checksummed, bounded batches over fully synchronous flushes to improve throughput, and paid for a recovery path that must validate and replay records.

**Results:**

- On a 12th-generation Intel Core i7-class laptop, a 100 MiB dataset with 16 KiB values and 100% unique keys reached 11,800 committed writes/s at p95 commit latency of 2.8 ms and p99 commit latency of 6.4 ms with a 64 MiB segment size and a 4 MiB compaction queue.
- With 16 concurrent repair clients and a 100 MiB index, MerkleDB transferred 2.7 GiB of candidate blocks in 41.2 s, bounded worker memory at 1.9 GiB, and completed with 0 unverified blocks.
- After a forced process termination during a 50 GiB checkpoint, replayed recovery took 3.6 s from a 64 MiB segment, and every committed sequence number through the last valid record was present in the final index.
- Under a workload with 80% reads, 15% point writes, and 5% range writes, a 10 GiB working set on a 12th-generation Intel Core i7-class laptop used 3.8 GiB of resident memory and held read p95 latency at 0.41 ms while compaction ran.

#### Slate

A small journaling block-storage layer that exposes append-only extents, checksums, and crash-replayable metadata for a local block device.

**Architecture:** Slate stores extents in 1 MiB data extents and maintains a 16 MiB metadata journal with fixed-size records, generation numbers, CRC-32C checksums, and a two-generation superblock. A single metadata coordinator serializes journal commits, while worker threads append data extents and maintain a bounded dirty-page queue. Recovery validates the current generation, replays committed metadata records, and promotes the new superblock only after the index is consistent. The host interface is a Unix socket protocol carrying extent IDs, byte offsets, operation types, lengths, and checksums; each request is framed and bounded to 4 MiB. The on-disk layout keeps metadata and data extents separated so metadata replay does not depend on data placement.

**Trade-offs:** I chose a journaling metadata log over in-place metadata updates to make recovery deterministic, and paid for additional metadata writes and space reserved for the two generations. I chose explicit extent IDs over a fully indexed free-space tree to keep allocation simple and bounded, and paid for an allocation scan that becomes more expensive as the device grows. I chose Unix-socket local transport over a network protocol to avoid network authentication and fragmentation concerns, and paid for a failure domain limited to one host.

**Results:**

- On a commodity NVMe drive with a 10 GiB device, 4 KiB writes, 100% unique extent IDs, and 8 concurrent appenders, Slate sustained 18,400 committed writes/s with p95 append latency of 1.9 ms and p99 append latency of 4.7 ms.
- After a forced power-loss simulation at 40% device utilization, metadata replay validated 184,320 records and promoted generation 1 in 2.4 s; generation 0 remained available for rollback.
- With a 10 GiB device and a 256 MiB dirty-page queue, a mixed workload of 70% reads, 25% random writes, and 5% metadata commits used 2.1 GiB of resident memory and held read p95 latency at 0.33 ms while compaction and journal flushes ran.
- A repair run over 100,000 extents with 8 concurrent readers verified all 100,000 extent checksums in 6.8 s and reported 0 corrupt extents.

### Technical Foundation

**Core Systems:** `Go`, `Go test`, `Go race detector`, `Go benchmark harness`, `Google Protocol Buffers`, `etcd client`

**Storage & Data:** `LSM trees`, `B+ trees`, `CRC-32C`, `Unix sockets`, `page-aligned block formats`

**Infrastructure & Observability:** `Prometheus`, `OpenTelemetry`, `systemtap`, `perf`, `Linux cgroups`

### How I Build

- I define invariant tests before implementation so every write path has an explicit check for sequence continuity, checksum validity, and bounded resource use.
- I cap queues and batches at fixed sizes so a slow disk or worker cannot turn a transient delay into unbounded memory growth.
- I make replay a first-class operation and test process termination, stale generations, and partial records instead of treating recovery as a startup edge case.
- I profile p50, p95, and p99 latency with a fixed workload and machine class, then change one parameter at a time so each optimization has a measurable cost.

### Current Explorations

- The Rust `Raft` paper studies how deterministic leader election, log replication, and membership changes can preserve a coherent state machine across failure domains.
- The `raft` RFC studies how configuration changes and log compaction interact with safety during membership transitions.
- The Linux `io_uring` kernel subsystem studies how prepared I/O and completion queues can reduce syscall overhead while preserving bounded submission and completion limits.

### Contact

[GitHub](https://github.com/albertinamcmenimen)