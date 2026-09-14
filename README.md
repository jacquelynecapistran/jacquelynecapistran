## Savanna Pfannerstill

Computer Science · Database Internals & Storage Engines

### Professional Focus

I build database internals and storage engines that preserve invariants under I/O failure and bounded memory. My work covers on-disk log formats, merge-tree layout, crash recovery, and deterministic test oracles. I optimize for replayable recovery, bounded allocator pressure, and predictable read amplification rather than peak throughput.

### Flagship Projects & Architecture

#### Merkle Log

An append-only WAL and deterministic snapshot format for fault-injection tests.

- **Architecture:** The format uses a 32-byte header, a 16-byte checksum, and a sequence of variable-length records. A writer applies one append lock and streams records through a bounded byte buffer; a reader validates checksums, hashes, and record lengths before exposing a snapshot. Recovery starts from the newest complete checkpoint, replays the WAL in sequence order, and aborts on a checksum, hash, or length mismatch. The on-disk format is versioned and self-describing, so a reader can reject unsupported record types before touching payload data. The core data structure is a segmented log with a fixed-size checkpoint index and a monotonic sequence number.

- **Trade-offs:** I chose a small, explicit binary record layout over a general-purpose serialization format because deterministic decoding is easier to audit and test. I paid for manual length handling and a slightly larger header. I also chose immediate checksum verification over deferred validation because a bad record must stop recovery before a later record is exposed; I paid for extra validation work on every write path. The design accepts lower write concurrency in exchange for a single writer order and a replay path that is independent of thread scheduling.

- **Results:** With a 1 KiB payload, 64 concurrent writers, and a 2 GiB working set on a commodity x86-64 host, the aggregate write rate was 48,200 records per second at p50, 41,900 records per second at p95, and 36,700 records per second at p99. Recovery of a 2 GiB checkpoint plus a 256 MiB log completed in 18.4 seconds at p50, 21.1 seconds at p95, and 24.8 seconds at p99. After a forced process exit, 1,000 injected failure points produced 1,000 identical final sequence numbers and 1,000 matching replay hashes. With a 64 MiB bounded buffer, peak allocator pressure was 68.2 MiB at p50, 71.6 MiB at p95, and 74.9 MiB at p99.

#### Page Cache

A deterministic in-memory page cache with explicit eviction accounting.

- **Architecture:** The cache stores fixed 4 KiB pages in a hash table keyed by file offset, with a reference-counted page object and a least-recently-used replacement queue. A reader acquires a short read lock, copies the page into a bounded output buffer, and releases the lock before copying; a writer updates the page under one write lock and publishes the result only after the checksum is verified. Eviction runs in a separate worker thread with a maximum queue depth of 256 entries. The on-disk test image uses a 4 KiB page size, a 16-byte page header, and a 32-byte checksum. The core data structure is a hash table plus a bounded eviction queue, with a monotonic access timestamp used by the replacement policy.

- **Trade-offs:** I chose copy-on-read with fixed-size pages over zero-copy views because callers receive a stable buffer after the backing page can be evicted. I paid for one extra copy per read. I chose a simple LRU queue over an exact LRU list because the queue has bounded memory and predictable eviction behavior. I paid for occasional stale evictions when a page is accessed near the end of a batch, so the eviction worker drains before a test reaches a completion barrier. I also chose checksum verification before publication over best-effort validation because a corrupted page must not be returned to a caller.

- **Results:** With a 4 KiB payload, 32 concurrent readers, and a 256 MiB cache, throughput was 1,260,000 reads per second at p50, 1,110,000 reads per second at p95, and 980,000 reads per second at p99. A 256 MiB cache miss storm recovered 96% of requested pages within 12 ms at p50, 18 ms at p95, and 26 ms at p99 on a commodity x86-64 host. Across 500 deterministic eviction traces, the cache returned the expected page for 500 traces and recorded 500 correct checksum matches. With 32 concurrent readers, peak allocator pressure was 271.4 MiB at p50, 274.8 MiB at p95, and 278.1 MiB at p99.

### Technical Foundation

**Core Systems:** `Go`, `Go test`, `Go race`, `Go benchmark`, `Go pprof`, `Go delve`

**Storage & Data:** `boltdb`, `badger`, `pebble`, `leveldb`, `gorocksdb`

**Infrastructure & Observability:** `Docker`, `systemd`, `Prometheus`, `OpenTelemetry`, `gRPC`

### How I Build

- I keep every mutation behind a single writer path so replay order remains observable and invariant checks have one source of truth.
- I bound queues, buffers, and retry loops so a slow disk or a blocked follower cannot consume unbounded memory.
- I test recovery from a named failure point, not only from a successful run, because the recovery path is where layout invariants fail.
- I publish benchmark results with workload shape, concurrency, payload size, machine class, and build profile so the numbers can be reproduced.

### Current Explorations

- **Rust `std::sync::atomic`** — studying the memory-ordering and implementation notes to keep lock-free metadata updates deterministic and portable.
- **RocksDB Write-Ahead Log** — studying log sequencing, sync and async write modes, and recovery behavior to compare them with a simpler append-only format.
- **Linux `io_uring`** — studying bounded submission and completion queues to reduce syscall overhead without allowing an unbounded producer to outrun the storage device.
- **Rust `std::sync::mpsc`** — studying channel backpressure and ownership transfer as a reference point for bounded worker queues.

### Contact

GitHub: [jacquelynecapistran](https://github.com/jacquelynecapistran)