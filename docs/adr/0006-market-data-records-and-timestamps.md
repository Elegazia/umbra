# ADR 0006: Market-data record schemas and timestamp policy

- Status: Accepted
- Date: 2026-11-15
- Deciders: Project owner
- Related: [ADR 0002](0002-instrument-identity-and-definition.md), [ADR 0005](0005-numeric-representation.md), ADR 0007, [ADR 0008](0008-latency-target-and-deployment-model.md)

## Context

Umbra records trades, order-book changes and candles from several venues and replays them through the same code that runs live. This ADR defines the normalised records those components exchange and store, and which timestamps they carry.

Earlier decisions fix much of the shape. [ADR 0008](0008-latency-target-and-deployment-model.md) requires fixed-width records that the C++ core can read in place. [ADR 0005](0005-numeric-representation.md) stores prices and quantities as 64-bit mantissas with 8-bit decimal exponents. [ADR 0002](0002-instrument-identity-and-definition.md) identifies instruments by a 64-bit `instrument_id`, resolved as of the event time.

Venues send book changes as messages that update a variable number of price levels at once. Kraken WebSocket v2 sends up to the subscribed depth per message with a CRC32 checksum over the top of the book; Binance sends batches of changed levels with first and last update ids; Coinbase sends level changes with a sequence number. A book must only be checked against a venue checksum after a whole message has been applied.

Venues also differ in how they timestamp events. Kraken sends RFC 3339 times with microsecond resolution, and Binance sends millisecond times by default, with a microsecond option. Some messages carry no event time at all.

Candles (klines) are needed for long-horizon research. Venues publish them, and data.binance.vision provides years of them in bulk, but venue candles cannot be checked without independent trade data.

## Decision drivers

Hard constraints:

- D1. Fixed width. Every record type has a fixed size and natural alignment, so it can be read in place by C++ and stored in a ring buffer (ADR 0008).
- D2. Lossless with respect to the venue. Every change the venue sent can be reconstructed, including message boundaries, so checksums can be verified.
- D3. Deterministic replay. Replaying the same raw input produces byte-identical records.

Preferences, in order:

- D4. Compact records, ideally within one 64-byte cache line for the most frequent type.
- D5. Simple translation to Parquet columns for research.
- D6. Candle history that can be validated.

## Considered options

### 1. Order-book updates

1A. One record per venue message, holding a variable-length list of levels.

- Good, because it mirrors the feed one to one (D2).
- Bad, because records are not fixed width (fails D1), and Parquet needs a nested list column (D5).

1B. One fixed-width record per changed level, with a flag on the last record of each venue message.

- Good, because records are fixed width (D1), and the flag preserves message boundaries, so the book is checked only after a complete message (D2).
- Good, because each record maps to one flat Parquet row (D5).
- Bad, because a message that changes many levels becomes many records, each repeating the header.

1C. A full copy of the top N levels after every update.

- Good, because queries for the book state at a given time read one row (D5).
- Bad, because storage grows by roughly a factor of N, and individual changes are lost, so checksums cannot be verified from the stored data (fails D2).

Databento's market-by-order schema uses the same design: one fixed-width record per change, with an `F_LAST` flag (bit value 0x80) on the last record of each venue event, and its documentation advises examining the book only after a record with that flag ([Databento order tracking example](https://databento.com/docs/examples/order-book/order-tracking)).

### 2. Timestamps in market-data records

2A. Each record carries the event time, the receive time and the time each processing stage finished.

- Good, because one record shows the full path of a message.
- Bad, because stage times differ on every replay, so the same raw input produces different records (fails D3), and the records grow by 16 or more bytes (D4).

2B. Each record carries the event time and the receive time. Stage times go to a separate latency log keyed by the record's sequence.

- Good, because records depend only on the market and on when the data arrived, so replay is deterministic (D3), and the header stays at 40 bytes (D4).
- Bad, because per-stage latency analysis joins the latency log to the records.

### 3. Candles

3A. Candles are derived only from recorded trades.

- Good, because they are consistent with Umbra's own data.
- Bad, because there is no candle history before recording began (D6).

3B. Only venue candles are stored.

- Good, because years of history are available immediately.
- Bad, because they cannot be checked against anything (D6).

3C. Both, with a flag marking candles that Umbra derived itself.

- Good, because bulk history comes from the venue, and derived candles over recorded periods validate it (D6).
- Bad, because there are two sources to reconcile, and queries have to choose one.

## Decision

Chosen: 1B, 2B and 3C.

Storing one fixed-width record per changed level is the only option that keeps records fixed width while preserving everything the venue sent; the last-in-message flag means checksums are verified at the right points. Stage times are kept out of the records because they describe Umbra's performance rather than the market, and keeping them out means the same raw data always produces the same records. Candles come from both sources because bulk history is only available from the venue, while candles derived from recorded trades are what make that history checkable.

### Encoding

All records are little-endian, packed with natural alignment, and a multiple of 8 bytes long. Every record starts with the same 40-byte header.

| Offset | Field | Type | Notes |
|---|---|---|---|
| 0 | `record_type` | u8 | 1 trade, 2 book update, 3 candle |
| 1 | `flags` | u8 | See below |
| 2 | `price_exp` | i8 | Exponent of every price in the record (ADR 0005) |
| 3 | `qty_exp` | i8 | Exponent of every quantity in the record |
| 4 | `length` | u16 | Size of the whole record in bytes |
| 6 | `version` | u8 | Layout version of this record type, starting at 1 |
| 7 | reserved | u8 | Zero |
| 8 | `instrument_id` | i64 | ADR 0002 |
| 16 | `ts_event` | i64 | Venue event time, ns since the Unix epoch (UTC) |
| 24 | `ts_recv` | i64 | Time Umbra received the bytes, ns since the Unix epoch (UTC) |
| 32 | `sequence` | u64 | Venue sequence or update id where provided, otherwise 0 |

Flags:

| Bit | Name | Meaning |
|---|---|---|
| 0 | `LAST` | Last record produced from one venue message |
| 1 | `SNAPSHOT` | Part of a full book snapshot rather than an incremental change |
| 2 | `NO_TS_EVENT` | The venue sent no event time; `ts_event` is 0 |
| 3 | `DERIVED` | Produced by Umbra (for example a candle built from trades), not sent by the venue |
| 4 | `SEQ_GAP` | The decoder detected missing venue messages before this record's message |
| 5–7 | reserved | Zero |

Trade (72 bytes):

| Offset | Field | Type | Notes |
|---|---|---|---|
| 40 | `price` | i64 | Mantissa |
| 48 | `qty` | i64 | Mantissa |
| 56 | `trade_id` | u64 | Venue trade id where numeric, otherwise 0 |
| 64 | `aggressor` | u8 | 0 unknown, 1 buyer, 2 seller |
| 65 | padding | 7 bytes | Zero |

Book update (64 bytes, one cache line):

| Offset | Field | Type | Notes |
|---|---|---|---|
| 40 | `price` | i64 | Mantissa |
| 48 | `qty` | i64 | New total quantity at this level; 0 removes the level |
| 56 | `side` | u8 | 1 bid, 2 ask |
| 57 | `action` | u8 | 1 set level, 2 clear the whole book (precedes a snapshot) |
| 58 | padding | 6 bytes | Zero |

Candle (88 bytes):

| Offset | Field | Type | Notes |
|---|---|---|---|
| 40 | `open`, `high`, `low`, `close` | 4 × i64 | Mantissas; `ts_event` is the candle's open time |
| 72 | `volume` | i64 | Mantissa, base-asset quantity |
| 80 | `interval_s` | u32 | Candle length in seconds |
| 84 | `trade_count` | u32 | Number of trades, or 0 if unknown |

In Parquet, each record type is a table with one column per field. Prices and quantities follow ADR 0005, with the record's `price_exp` and `qty_exp` as columns.

### Rules

1. Raw messages are the source of truth. The recorder stores every message exactly as received, with its `ts_recv` and connection, and normalised records are derived from them. Any record table can be rebuilt from the raw log.
2. `ts_event` and `ts_recv` are wall-clock times (`CLOCK_REALTIME`), kept synchronised with chrony, so that they can be compared with each other and across machines. Stage latencies in the latency log use the monotonic clock (ADR 0008). The two clocks are never mixed in one calculation.
3. Venue times are converted to nanoseconds exactly: milliseconds are multiplied by 10^6 and microseconds by 10^3. They are never parsed through floating point.
4. During replay, `ts_recv` is the receive time stored in the raw log, never the time of the replay.
5. A venue message that changes n levels becomes n book-update records in the venue's order, and only the last carries `LAST`. A snapshot is a clear record followed by set records, all flagged `SNAPSHOT`.
6. Candles sent by the venue and candles derived by Umbra are stored in the same table and distinguished by the `DERIVED` flag. A derived candle covers only trades that Umbra recorded for the whole interval; an interval with a recording gap produces no derived candle.
7. Readers check `length` and `version` before interpreting a record. A record with an unknown version is rejected, and `length` lets a reader skip a record type it does not handle. A change to a layout increments its version; it never reuses a version number.
8. Where a venue numbers its messages, the decoder checks that each message follows the previous one. For Binance diff-depth updates, which carry a first update id `U` and a last update id `u`, a message is contiguous when its `U` is the previous message's `u` plus one; `sequence` holds `u`, and `U` remains available in the raw log. When a message is not contiguous, every record produced from it carries `SEQ_GAP`, and the order book treats itself as invalid until it has applied a new snapshot.

## Consequences

The C++ core reads records in place and moves them through ring buffers without conversion. The most frequent record, the book update, fits in one cache line. Replays of the same raw data produce identical records, which makes them usable as regression tests.

A message that changes many levels produces many records, each with a 40-byte header. Research queries that need the book state at a point in time have to rebuild it from updates, or use snapshots produced separately.

Latency analysis joins the latency log to the records by `sequence` and `instrument_id`.

The validation milestone can compare venue candles with derived candles over every recorded period, and report differences per interval.

Follow-up work:

- ADR 0007 defines how sources deliver raw messages and records.
- The recorder's raw log format is defined when the recorder is built.
- A quote (best bid and offer) record is added if a source is used whose best-price stream carries information that book updates do not.

### Revisit when

- more than 1% of the venue messages for a source change more than about 20 levels, so that per-level headers dominate storage (consider a compact multi-level record for that source);
- a venue's sequence or update id does not fit in 64 bits;
- a venue in scope provides order-level (market-by-order) data that Umbra uses (add an order record type);
- `trade_id` values for a venue are not numeric and are needed outside the raw log (add a mapping or a wider id).

## Background

### Market by price and market by order

A market-by-price feed reports the total quantity at each price level; a market-by-order feed reports each individual order. Most crypto venues publish market-by-price data. Reconstructing a book means applying each level change in order, starting from a snapshot, and checking the result against the venue's sequence numbers or checksum.

### Event time and receive time

The event time is when something happened at the venue. The receive time is when Umbra saw it. The difference between them includes network delay and any delay at the venue, and comparing receive times across venues is how lead–lag effects are measured. Receive times are only comparable if the clocks that produce them are synchronised.

### Wall clock and monotonic clock

A wall clock tells the time of day and can be adjusted, including backwards, when it is synchronised. A monotonic clock only moves forwards and is suited to measuring intervals, but has no relation to the time of day. Timestamps that are compared with venue times have to come from the wall clock; durations inside the process have to come from the monotonic clock.

### Fixed-width records

Records of a fixed size can be read by casting a pointer to a structure, stored back to back in arrays and ring buffers, and indexed by position. Natural alignment, where each field starts at a multiple of its own size, avoids slow unaligned reads on some processors and keeps the layout the same across compilers.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Kraken WebSocket v2: book checksum guide](https://docs.kraken.com/api/docs/guides/spot-ws-book-v2) | Primary | 2026-11-15 |
| [Databento: order tracking example (`F_LAST`)](https://databento.com/docs/examples/order-book/order-tracking) | Primary | 2026-11-15 |
| [dbn: record flags](https://docs.rs/crate/dbn/0.59.0/source/src/flags.rs) | Primary | 2026-11-15 |
| [Binance: WebSocket streams (`timeUnit=MICROSECOND`, managing a local order book)](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams) | Primary | 2026-11-15 |
| [dbn: record header (`length` field)](https://docs.rs/crate/dbn/0.59.0/source/src/record.rs) | Primary | 2026-11-15 |
| [Binance SBE stream schema](https://developers.binance.com/en/docs/products/spot/sbe/schemas/stream_1_0.xml) | Primary | 2026-11-15 |
| [Linux `clock_gettime(2)` (realtime and monotonic clocks)](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) | Primary | 2026-11-15 |
| [chrony documentation](https://chrony-project.org/documentation.html) | Primary | 2026-11-15 |
