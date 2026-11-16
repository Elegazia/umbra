# ADR 0008: Latency target and deployment model

- Status: Accepted
- Date: 2026-11-15
- Deciders: Project owner
- Related: [ADR 0001](0001-brokers-venues-and-data-sources.md), [ADR 0002](0002-instrument-identity-and-definition.md), ADR 0005, ADR 0006, ADR 0007

## Context

ADRs 0005 (numeric representation), 0006 (record schemas) and 0007 (`DataSource` interface) each trade simplicity against speed. This ADR fixes what speed Umbra is designed for and how it is measured, so that those decisions have a common reference.

Umbra's hot path is designed to the constraints of a system colocated with an exchange. In that setting the network to the matching engine costs microseconds, so the time spent inside the process dominates. A single JSON parse in Python takes tens of microseconds, which is more than the whole budget. The hot path therefore has to be compiled code that does not allocate per message. This is a design requirement of the project and applies whether or not a particular deployment is colocated.

Umbra is actually run on a workstation or a rented virtual server, where the round trip to a venue is measured in milliseconds. Measuring in-process latency against a live connection would mostly measure the network. In-process latency is therefore measured by replaying recorded market data through the same code that runs live.

Venues deliver market data in two encodings. Most crypto venues send JSON over WebSocket. Exchange colocation feeds are binary (for example Nasdaq ITCH and CME MDP 3.0), and some crypto venues also offer binary streams: Binance publishes SBE-encoded spot market data on a separate endpoint. Umbra is intended to be usable for live trading, so it has to handle JSON venues as they are, while still being measurable against a binary feed.

## Decision drivers

Hard constraints:

- D1. The hot path meets colocation-grade constraints: compiled code, integer arithmetic, no allocation per message, and timing per stage.
- D2. The same core can trade live on a JSON venue without an offline conversion step.
- D3. Published latency figures can be reproduced on the stated hardware.

Preferences, in order:

- D4. Figures use standard terms (tick-to-trade, percentiles) and can be compared with published numbers.
- D5. Measurement captures tail latency rather than averaging it away.
- D6. Paid infrastructure is used only while benchmarking.

## Considered options

The decision has three parts.

### 1. Latency target

Targets are for internal tick-to-trade in replay, as defined under Decision.

1A. p99 ≤ 50 µs.

- Good, because it is reachable with straightforward C++.
- Bad, because it permits per-message allocation and similar habits that D1 rules out.

1B. p99 ≤ 10 µs and p99.9 ≤ 50 µs.

- Good, because it requires the main techniques (no allocation, integer prices, a SIMD JSON parser) while leaving room to parse JSON on the hot path (D2).
- Bad, because a JSON-inclusive figure is not comparable with binary-feed systems (D4).

1C. p99 ≤ 2 µs.

- Good, because it is close to published figures for tuned software systems on binary feeds (D4).
- Bad, because it is not achievable with JSON parsing inside the budget, and needs pinned, isolated cores on a tuned Linux host.

### 2. Feed encoding

2A. JSON only. The core parses venue JSON on the hot path.

- Good, because one decoder serves every JSON venue (D2).
- Bad, because parsing dominates the budget and the figures cannot be compared with binary-feed systems (D1, D4).

2B. Binary only. Recorded JSON is converted offline into fixed-width records, which the core replays.

- Good, because it matches how colocated feeds are consumed (D1, D4).
- Bad, because live trading on a JSON venue would need a conversion step in front of the core (fails D2).

2C. Two decoders, JSON and binary, that both produce the same internal records. The decoder is chosen per source.

- Good, because binary mode gives a comparable figure and JSON mode lets the same core trade on JSON venues (D1, D2, D4).
- Bad, because there are two decoders to write and keep consistent.

### 3. Benchmark host

3A. The development machine (macOS).

- Good, because it is free and always available (D6).
- Bad, because macOS does not support binding a thread to a core (affinity tags are scheduling hints only), cannot reserve cores away from the scheduler, and laptop clock speeds vary with temperature and power. Results are not reproducible (fails D3).

3B. Development on macOS, with published figures taken on a Linux virtual server with dedicated cores.

- Good, because Linux supports core binding (`sched_setaffinity`, `taskset`) and core isolation (`isolcpus`), which makes runs repeatable (D3).
- Bad, because benchmark runs need a paid server (D6), and virtual servers can still be affected by other tenants.

## Decision

Chosen: 1B and 1C as per-mode targets, 2C, and 3B.

Umbra has one core and two decoders. A source supplies bytes, a decoder turns them into the fixed-width internal records defined in ADR 0006, and the order book, signal and order stages are shared by both modes.

| Mode | Decoder | Use | Target |
|---|---|---|---|
| JSON | SIMD JSON parser over messages as received | Live trading and recording on JSON venues | p99 ≤ 10 µs, p99.9 ≤ 50 µs |
| Binary | Fixed-width records read in place (a venue's binary stream such as Binance SBE, or Umbra's own replay format) | Binary venues; colocation-grade benchmark | p99 ≤ 2 µs |

Both targets are provisional and will be reviewed after the first replay measurements.

Both modes are needed because Umbra is meant to trade, not only to be benchmarked. Binary mode is what a colocated system would consume, and it gives a figure that can be compared with such systems. JSON mode exists because the venues Umbra can trade on publish JSON, and the core has to work with their feeds as they are. Option 2C is the only encoding option that covers both (D1, D2).

Each mode has its own target so that the cost of parsing is counted only where it is actually paid. Option 3B is the only host that gives reproducible figures (D3).

### Measurement

Internal tick-to-trade is the time from a message's bytes becoming available in the receive buffer to the resulting order's bytes being passed to the send call. In replay, the bytes become available when the harness writes them to the buffer. A message that does not lead to an order is timed to the end of the last stage it reaches.

The decode, normalise, book update (including checksum), signal and order-encode stages are timed separately, and an end-to-end figure is reported as well. Reports give p50, p99, p99.9 and the maximum, together with the number of messages. Means and standard deviations are not used as headline figures.

Samples are recorded in an HDR histogram. Timestamps come from `std::chrono::steady_clock`. The cost of reading the clock is measured and reported; a cycle counter is used instead only if that cost is a significant fraction of a stage.

Each replayed message carries its scheduled send time, and latency is measured from that time. If the system stalls, messages queued behind the stall are recorded as late, not as fast (see Coordinated omission, below).

A published run records the CPU model, core count, kernel version, isolated cores and build flags. A result counts as reproducible if a re-run on the same configuration gives each percentile within 10%.

## Consequences

Performance claims become specific and can be re-run with one command. Supporting a new venue that has a binary stream only requires a new decoder.

Two decoders have to be maintained. They are tested against each other: the same recorded messages, passed through each decoder, must produce identical internal records.

The hot path is C++ in the target design. Python remains the reference implementation and the research layer.

This ADR constrains the following decisions:

- ADR 0005 must use integers for prices and quantities on the hot path.
- ADR 0006 must define fixed-width records that can be read in place, and a way to time each stage of each record.
- ADR 0007 must define an interface that can be implemented in C++, without depending on Python objects or `asyncio`, and with the decoder as a separate component.

Replay leaves out the network stack and tends to run with warm caches, so it overstates live performance. Replay figures are always labelled as internal tick-to-trade in replay, never as end-to-end latency. Interference from other tenants on a virtual server is reduced by using dedicated-core instances and repeating runs; the 10% reproducibility check detects it when it happens.

Two further ADRs follow from this one: how the C++ core is built and bound to Python, and which hosting region to use if Umbra runs near a venue.

### Revisit when

- the first replay measurements miss a target by more than a factor of 2 after profiling (supersede with a target based on the measurements);
- the targets are met with more than 5 times headroom (tighten them);
- a venue that ADR 0001 allows for trading offers binary market data (use binary mode live there);
- Umbra is deployed in or next to a venue's data centre (add network-inclusive measurement with hardware timestamps and PTP clock synchronisation, and redefine tick-to-trade accordingly).

## Background

### Tick-to-trade

Tick-to-trade is the usual latency figure for a trading system: the time from a market-data packet arriving to the order it triggers leaving. Colocated firms measure it at the network card using hardware timestamps. Tuned software systems are generally reported in the low single-digit microseconds, and FPGA-based systems below one microsecond. Umbra measures only the part of this path that runs inside the process.

### Percentiles

Latency distributions have a long right tail, caused by cache misses, page faults, context switches, interrupts and, in managed languages, garbage collection. A mean describes the bulk of the distribution and hides the tail. In trading, the tail decides who reacts first, so figures are given as percentiles: p99.9 is the latency exceeded by one message in a thousand.

### Coordinated omission

A benchmark that sends each message only after the previous one has been processed slows down exactly when the system stalls. The messages that would have arrived during the stall are sent late and measured as fast, so the stall barely appears in the results. Gil Tene named this effect coordinated omission and described the correction: measure from the time each message was meant to be sent. HdrHistogram includes a recording mode for this.

### Colocated system design

When the network costs microseconds, the operating system and the CPU become the bottleneck. Colocated systems therefore read binary feeds in place instead of parsing text, and bypass the kernel network stack using tools such as OpenOnload or DPDK. They run one thread per stage, pinned to an isolated core and polling rather than sleeping, because waking a sleeping thread takes microseconds. They avoid allocation, lay data out to fit cache lines, pass data between threads through lock-free single-producer single-consumer ring buffers, and use integer prices. Writing to disk is left to a separate process so the trading path never waits on I/O.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Binance: SBE market data streams](https://developers.binance.com/docs/binance-spot-api-docs/sbe-market-data-streams) | Primary | 2026-11-15 |
| [Gil Tene: How NOT to Measure Latency (QCon SF 2012 slides)](https://qconsf.com/sf2012/dl/qcon-sanfran-2012/slides/GilTene_HowNotToMeasureLatency.pdf) | Primary | 2026-11-15 |
| [HdrHistogram](https://hdrhistogram.github.io/HdrHistogram/) | Primary | 2026-11-15 |
| [simdjson](https://simdjson.org/) | Primary | 2026-11-15 |
| [Linux kernel parameters (`isolcpus`)](https://docs.kernel.org/admin-guide/kernel-parameters.html) | Primary | 2026-11-15 |
| [Apple: Thread Affinity API release notes](https://developer.apple.com/library/archive/releasenotes/Performance/RN-AffinityAPI/) | Primary | 2026-11-15 |
| [cppreference: `std::chrono::steady_clock`](https://en.cppreference.com/w/cpp/chrono/steady_clock) | Primary | 2026-11-15 |
| [NautilusTrader adapter survey, section 2](../nautilus-adapter-survey.md) | Secondary | 2026-11-07 |
