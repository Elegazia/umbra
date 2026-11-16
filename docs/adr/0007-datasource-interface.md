# ADR 0007: `DataSource` interface

- Status: Proposed
- Date: 2026-11-16
- Deciders: Project owner
- Related: [ADR 0002](0002-instrument-identity-and-definition.md), [ADR 0006](0006-market-data-records-and-timestamps.md), [ADR 0008](0008-latency-target-and-deployment-model.md)

## Context

A data source is everything between a venue and Umbra's core: the network connection, the subscriptions, the bytes the venue sends, and the work of turning them into the records defined in [ADR 0006](0006-market-data-records-and-timestamps.md). The same core also has to run against recorded data, where the source is a file.

Earlier decisions constrain the interface. [ADR 0008](0008-latency-target-and-deployment-model.md) requires that it can be implemented in C++, that it does not depend on Python objects or `asyncio`, and that the decoder is a separate component, so that JSON and binary decoders can be swapped per source. ADR 0006 makes the raw messages the source of truth: every message is logged as received, with its receive time, and replaying the log must produce identical records. ADR 0006 also says that after a sequence gap the order book is invalid until a new snapshot has been applied.

Venues differ in how a book is started and repaired. Binance sends incremental depth updates on a WebSocket and the starting snapshot over REST, and a client has to buffer updates, fetch the snapshot and discard updates older than it. Kraken sends the snapshot on the same WebSocket subscription. After a disconnect, every venue requires a new snapshot.

Network input arrives at times Umbra does not control, while the hot path is designed to poll without sleeping on its own core (ADR 0008). The interface decides where those two meet.

## Decision drivers

Hard constraints:

- D1. Implementable in C++ with no dependence on Python objects or `asyncio` (ADR 0008).
- D2. The decoder is a separate, swappable component (ADR 0008).
- D3. Deterministic replay: the core cannot tell a replayed source from a live one, and the same raw log produces the same records (ADR 0006).
- D4. Every raw message the core sees, including snapshots fetched over REST, is in the raw log.

Preferences, in order:

- D5. The hot path never waits on network or disk I/O and does not allocate per message.
- D6. Venue-specific behaviour, such as how to resynchronise a book, stays out of the core.
- D7. A Python reference implementation can follow the same shapes without much extra code.

## Considered options

The decision has three parts.

### 1. Control flow

1A. Push. The source calls a handler on the core, such as `on_message(bytes)`, whenever data arrives.

Used by cryptofeed and by NautilusTrader's data clients, and offered by Databento's threaded C++ client.

- Good, because it is the natural style for `asyncio` and callback-based network libraries.
- Bad, because the source decides when core code runs, so the interleaving of messages and timers depends on the network, which makes replay harder to keep identical to live (D3).
- Bad, because it does not fit a core that owns its thread and polls (D5).

1B. Pull. The core calls `poll()` on each source, which returns the next message or nothing, without blocking.

Offered by Databento's C++ client as `TryNextRecord()`, which returns the next buffered record without performing I/O, or `nullptr`.

- Good, because one thread owns the loop, and a replay source is simply a source whose `poll()` reads from a file (D3).
- Good, because it matches a polling core (D5).
- Bad, because a live source still has to receive data somewhere between polls, and the option does not say where.

1C. Pull over a ring buffer. As 1B, but each live source receives on its own thread, which writes complete messages and their receive times into a single-producer single-consumer ring buffer. `poll()` reads from the ring.

hftbacktest uses a similar split for live trading: a separate connector process per venue handles WebSocket and REST traffic and passes data to the trading process through shared memory.

- Good, because it has the interface and determinism of 1B (D3), and network stalls, TLS and reconnects happen on a thread the hot path never waits for (D5).
- Bad, because it needs a lock-free ring buffer, a second thread per source, and a policy for a full ring.

### 2. Decomposition

2A. One object per venue that connects, subscribes, decodes and resynchronises.

- Good, because each venue is in one place and there are few types.
- Bad, because decoding is bound to the connection, so a venue cannot switch between JSON and binary decoders, and a decoder cannot be tested without a connection (fails D2).

2B. Three components per source:

- Transport: connects, reconnects, receives complete messages and writes them, with receive times, into the ring.
- Decoder: turns one raw message into zero or more ADR 0006 records, on the hot path.
- Session: subscriptions, heartbeats, snapshot requests and resynchronisation, on the cold path.

- Good, because decoders are swappable and can be tested on recorded bytes alone (D2), and transports can be shared between venues that use the same protocol.
- Good, because the code that runs on the hot path, the decoder, is separated from code that may block or allocate.
- Bad, because there are more types, and the three have to agree on the frame format and on a channel between the core and the session.

### 3. Recovery after a gap

3A. The session repairs the book. When the decoder reports a gap or the transport reconnects, the session fetches a snapshot and passes it into the same ring as the stream. The decoder turns it into clear and snapshot records as ADR 0006 already defines.

- Good, because the order book only ever applies records and never needs to know which venue they came from (D6).
- Good, because the snapshot travels through the ring, so it is in the raw log and replay repeats the recovery (D4, D3).
- Bad, because the core needs a channel back to the session to report the gap.

3B. The consumer repairs the book. The order book or strategy requests a snapshot from the source.

- Bad, because each venue's recovery procedure, such as Binance's rules for discarding stale updates, ends up in the core (fails D6).
- Bad, because a snapshot requested outside the stream may not be logged in order with it (D4).

## Decision

Chosen: 1C, 2B and 3A.

A pull interface is what lets one thread own the hot path and lets a recorded file stand in for a venue. Putting a ring buffer behind it keeps network I/O on another thread, so the core polls without ever waiting for the network. The three-part split is the form ADR 0008's separate decoder takes once connection handling and recovery are also given a place. Recovery belongs to the session because it is specific to each venue, and sending the snapshot through the ring keeps the raw log complete, so that a replay goes through the same recovery as the live run did.

### Frames

A transport writes frames into the ring. A frame is one complete venue message, after TLS and WebSocket framing have been removed, with a small header:

| Field | Type | Notes |
|---|---|---|
| `ts_recv` | i64 | Receive time, ns since the Unix epoch (UTC), as in ADR 0006 |
| `length` | u32 | Number of payload bytes |
| `connection_id` | u16 | Identifies the connection within the source |
| `kind` | u8 | 1 stream message, 2 snapshot, 3 connected, 4 disconnected |
| reserved | u8 | Zero |
| payload | `length` bytes | The message exactly as received; empty for kinds 3 and 4 |

The ring holds frames back to back. Frames of kind 2 carry a snapshot as the venue returned it, for example the body of a REST response. Frames of kinds 3 and 4 tell the core about connection changes.

### Interfaces

The interfaces are described here in language-neutral terms.

- `Source.poll()` returns the next frame, or nothing if none is ready. It never blocks. The returned frame is valid until the next call to `poll()` on that source.
- `Decoder.decode(frame, out)` writes zero or more ADR 0006 records for one frame into a caller-provided buffer and returns how many it wrote. It does not allocate. It resolves venue symbols through a lookup table built from the ADR 0002 registry before the hot path starts, and it checks sequence contiguity as ADR 0006 rule 8 describes.
- The session runs on the transport's thread or its own, never on the core's thread. It receives gap reports from the core through a second single-producer single-consumer ring running in the opposite direction.

A replay source implements `poll()` by reading frames from the raw log. A live source's transport is the only producer of its ring, and the core is the only consumer.

### Rules

1. The raw log contains exactly the frames the core consumed, in the order it consumed them, including snapshot and connection frames. The recorder reads them from its own ring, which the transport writes alongside the core's ring; the core thread never writes to disk.
2. When a decoder reports a gap, or a frame of kind 4 arrives, the order book for the affected instruments is invalid. The session fetches a snapshot and writes it as a frame of kind 2. Venue-specific rules, such as discarding Binance updates older than the snapshot, are applied by the decoder for that venue.
3. If the core's ring is full, the transport does not overwrite or silently drop frames. It closes the connection and writes a frame of kind 4 once there is space, so that the session resynchronises from a snapshot. Each occurrence is counted and reported.
4. If the recorder's ring is full, the recording has a gap. The gap is recorded with its time range, and the affected intervals produce no derived candles (ADR 0006 rule 6).
5. When several sources are replayed together, frames are merged in `ts_recv` order, with ties broken by source and `connection_id`.
6. The Python reference implementation keeps the same shapes: `poll()` returns a frame or `None` without blocking. Its transport may use `asyncio` internally on its own thread, with a thread-safe queue in place of the ring; no coroutine appears in the interface.

## Consequences

The core is a loop that polls its sources, decodes frames and applies records, and it is the same loop in live trading and in replay. Each decoder can be tested on recorded frames without a network, and the JSON and binary decoders for a venue can be checked against each other as ADR 0008 requires.

Every live source costs a thread, and the transport thread does TLS, WebSocket deframing and timestamping. Receive times are taken on that thread when a complete message has been read, so they include deframing but not the wait in the ring; the time spent in the ring is visible as the difference between `ts_recv` and the start of decoding.

Live processing order across several sources depends on the order in which the core polls them, while replay merges them by `ts_recv`. A replay therefore reproduces the order in which messages arrived, not necessarily the order in which a live run handled two messages that arrived within one polling cycle of each other.

Order entry is not covered by this ADR. Sending orders, and whether they share a connection with market data, is decided with the order path.

Follow-up work:

- The ring buffer is implemented and benchmarked as its own component in the C++ milestone.
- The raw log's on-disk format is defined when the recorder is built, and stores frames as defined here.

### Revisit when

- more than one consumer needs the same frames in one process, such as two strategies (consider a broadcast ring, or shared memory between processes as hftbacktest does);
- the number of live connections exceeds the number of available cores (consider one I/O thread serving many connections with `epoll`);
- the core's ring fills during normal operation rather than only during stalls (size the ring from measured bursts, or move decoding to the transport thread);
- a venue needs the core to send messages on the market-data connection, beyond subscriptions handled by the session.

## Background

### Push and pull interfaces

In a push interface the producer calls into the consumer; in a pull interface the consumer asks the producer for the next item. Push is convenient when the producer is driven by external events, such as a network library, but it hands control of timing to the producer. Pull keeps control with the consumer, which can then decide its own order of work, poll several inputs in a fixed order, and be driven by a recorded file instead of a live connection. A queue between the two is how a push-driven producer is turned into a pull interface.

### Single-producer single-consumer ring buffers

A ring buffer is a fixed array used as a queue, with a write index advanced by the producer and a read index advanced by the consumer. When exactly one thread writes and one thread reads, each index has only one writer, so no lock is needed: the producer publishes data by storing the write index with release ordering, and the consumer reads it with acquire ordering, which guarantees it sees the data written before the index. The two indices are kept on separate cache lines so that the threads do not repeatedly invalidate each other's caches (false sharing), and the capacity is usually a power of two so that positions wrap with a bit mask.

### Book synchronisation from a snapshot and a stream

A venue that publishes incremental book updates gives each update a sequence number or update id. A client starts by receiving updates, fetches a snapshot that states the id it is valid at, discards updates older than the snapshot, and applies the rest in order. Binance documents this procedure for its diff-depth stream: the first update applied must have a first update id at or below the snapshot's id plus one, and a last update id above it. Any break in the ids afterwards means an update was missed, and the process starts again from a new snapshot.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Databento C++ client: `live_blocking.hpp` (`NextRecord`, `TryNextRecord`)](https://github.com/databento/databento-cpp/blob/main/include/databento/live_blocking.hpp) | Primary | 2026-11-16 |
| [Databento C++ client: README (threaded client with callbacks)](https://github.com/databento/databento-cpp) | Primary | 2026-11-16 |
| [hftbacktest: live bot (`live/bot.rs`)](https://docs.rs/crate/hftbacktest/0.9.4/source/src/live/bot.rs) | Primary | 2026-11-16 |
| [hftbacktest: live trading guide (connector and shared memory)](https://www.mintlify.com/nkaz001/hftbacktest/guides/live-trading) | Secondary | 2026-11-16 |
| [cryptofeed](https://github.com/bmoscon/cryptofeed) | Primary | 2026-11-16 |
| [NautilusTrader: adapters](https://nautilustrader.io/docs/latest/concepts/adapters/) | Primary | 2026-11-16 |
| [Binance: WebSocket streams (managing a local order book)](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams) | Primary | 2026-11-16 |
| [rigtorp/SPSCQueue](https://github.com/rigtorp/SPSCQueue) | Primary | 2026-11-16 |
