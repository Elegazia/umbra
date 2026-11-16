# ADR 0005: Numeric representation of prices and quantities

- Status: Accepted
- Date: 2026-11-15
- Deciders: Project owner
- Related: [ADR 0002](0002-instrument-identity-and-definition.md), [ADR 0006](0006-market-data-records-and-timestamps.md), [ADR 0008](0008-latency-target-and-deployment-model.md)

## Context

Every price and quantity Umbra handles is compared, validated against a tick grid, hashed into order-book checksums and, later, sent in orders. All of these need exact arithmetic. Binary floating point cannot represent most decimal fractions exactly: in Python, `65000.3 % 0.1` evaluates to `0.09999999999930215` rather than 0, so a valid price fails a tick check. `Decimal` is exact, but every value is a heap-allocated object, which [ADR 0008](0008-latency-target-and-deployment-model.md) rules out on the hot path.

The usual answer is fixed-point: a value is stored as an integer mantissa m with a decimal exponent e, so that the value is m × 10^e. Values with the same exponent are added, subtracted and compared as plain integers. Two choices remain: whether the exponent is fixed for the whole system or varies, and how wide the integer is.

The range of a 64-bit signed integer is about 9.22 × 10^18. At a fixed exponent of −9 this leaves about 9.2 × 10^9 units, which is enough for any price but not for quantities of low-priced tokens, where single trades of more than 10^10 units occur. The opposite limit also occurs: NautilusTrader's own test data includes a Kraken futures contract with a tick of 10^−10, which its standard 10^−9 build rejects ([NautilusTrader adapter survey, section 1](../nautilus-adapter-survey.md)).

Venues are not consistent either. CME's MDP 3.0 and Databento use a fixed 10^−9. Binance and Bybit send SBE messages in which each price and quantity is a 64-bit mantissa with an 8-bit exponent carried in the message. Kraken's order-book checksum is computed from prices and quantities written with exactly the number of decimals the venue sends, with the decimal point and leading zeros removed.

## Decision drivers

Hard constraints:

- D1. Exact. Equal values compare equal, and tick-grid checks give the right answer.
- D2. Range. Every price, quantity and tick of the venues in ADR 0001 is representable, including tokens traded in quantities above 10^10 and ticks finer than 10^−9.
- D3. Hot-path cost meets ADR 0008: no allocation, single-instruction comparisons within an instrument.

Preferences, in order:

- D4. Venue values can be stored without conversion, so binary feeds can be copied in place and checksums reproduced from stored values.
- D5. Compact records.
- D6. Straightforward storage and querying in Parquet and DuckDB.

## Considered options

### A. Fixed exponent −9, 64-bit mantissa

Used by Databento, CME MDP 3.0 (`PRICE9`) and NautilusTrader's standard build.

- Good, because every value in the system shares one scale, so any two values compare directly (D3).
- Good, because each value takes 8 bytes (D5), and it maps to Parquet's `DECIMAL(18, 9)` stored as `INT64` (D6).
- Bad, because quantities above about 9.2 × 10^9 overflow and ticks finer than 10^−9 cannot be represented (fails D2).
- Bad, because venue values with a different number of decimals have to be rescaled on the way in, and back again to compute a checksum (D4).

### B. Fixed exponent −16, 128-bit mantissa

Used by NautilusTrader's high-precision build (`FIXED_PRECISION = 16`, `i128`), which its official Python packages ship.

- Good, because the range is ample in both directions (D2) and values still compare directly.
- Bad, because each value takes 16 bytes (D5), and 128-bit arithmetic uses two registers per value, which is slower on the hot path (D3).
- Bad, because Parquet needs `DECIMAL(38, 16)` stored as fixed-length bytes, and DuckDB needs `HUGEINT` (D6).
- Bad, because venue values still have to be rescaled (D4).

### C. 64-bit mantissa with an 8-bit exponent stored with each value

Used by Binance SBE and Bybit SBE. The exponent for an instrument is the number of decimals the venue uses for that field, so a value is stored exactly as the venue expresses it.

- Good, because the exponent adapts to each instrument, so both very large quantities and very fine ticks fit (D2).
- Good, because within one instrument all values share an exponent, so comparison is a single integer instruction (D3).
- Good, because binary feeds with mantissa-exponent fields are copied without conversion, and the decimal string of a stored mantissa is exactly the digit string a venue checksum is computed from (D4).
- Good, because each record carries its own exponent, so it stays readable if the venue later changes its precision.
- Bad, because comparing values from different instruments requires rescaling one of them by a power of ten.
- Bad, because each value takes one extra byte (D5), and Parquet stores the mantissa and exponent as two columns, with a view computing the decimal value (D6).

## Decision

Chosen: option C. Prices and quantities are stored as a signed 64-bit mantissa and a signed 8-bit decimal exponent, value = mantissa × 10^exponent.

C is the only option that meets D2 without paying for 128-bit arithmetic on the hot path, and it matches the format that venues with binary feeds already use. Keeping the venue's own number of decimals also means Kraken's order-book checksum can be computed directly from stored values, without formatting floats or rescaling.

### Rules

1. The exponent of a field is the instrument's published precision for that field, negated, taken from its definition (ADR 0002), for example Kraken's `price_precision` and `qty_precision`. It is not inferred from the digits of each message, because JSON numbers can drop trailing zeros. The decoder rescales each value to that exponent exactly, and rejects a value with more decimals than the precision allows. Values are never normalised: with a quantity precision of 8, `0.001` is stored as mantissa 100000 with exponent −8, not as 1 with exponent −3.
2. Decoders never parse prices or quantities through a floating-point type. JSON numbers are read as their raw text and converted to a mantissa and exponent directly.
3. Within one instrument, the hot path compares and adds mantissas directly. If the exponents of two values differ, the one with the larger exponent is rescaled first. Rescaling that would lose digits is an error.
4. Products such as price × quantity use a 128-bit intermediate and are rescaled to a stated result exponent. Overflow is an error, never a wrap-around.
5. Python works with these values as `(int, int)` pairs or exact `Decimal`, never `float`. The research layer may convert to `float64` at query time for statistics such as returns and volatility, where approximation is acceptable. Tick validation, order books and orders always use exact values.
6. In Parquet, each price and quantity is stored as two columns, `<name>_mantissa` (`INT64`) and `<name>_exp` (`INT8`). A DuckDB view exposes the decimal value.

## Consequences

Binary-feed decoders copy mantissas and exponents straight into Umbra's records. JSON decoders do the same work from text without going through floating point. Checksums are computed from stored values.

Cross-instrument work, such as comparing BTC prices on two venues for lead–lag research, rescales one side first. This happens in signal and research code, not in the order book.

The record layouts in ADR 0006 carry one price exponent and one quantity exponent per record, since every value in a record belongs to one instrument.

Every arithmetic path needs overflow checks, and tests cover the boundaries: the largest quantities, the finest ticks and mixed exponents.

### Revisit when

- a venue in scope sends values whose mantissa exceeds 64 bits at its own precision (consider a 128-bit mantissa for that field);
- rescaling between instruments shows up as a significant cost in profiles of the signal path (consider a common exponent per asset);
- the storage layout (ADR 0004) makes two columns per value impractical (consider a decimal logical type per column with a fixed scale per table).

## Background

### Why floating point fails here

A binary floating-point number is m × 2^e. Most decimal fractions, such as 0.1, have no finite representation in base 2, so they are stored as the nearest binary value. Arithmetic on these approximations accumulates error, which is why `0.995 % 0.005` evaluates to `0.004999…` in Python. For statistics this error is negligible. For deciding whether a price is valid or whether two book levels are the same price, it is wrong.

### Fixed-point arithmetic

A fixed-point number is m × 10^e with an integer m. With a shared e, addition, subtraction and comparison are integer operations and are exact. Multiplication adds the exponents and multiplies the mantissas, which can overflow even when both inputs fit, so products are computed in a wider type. Division is not exact in general and needs an explicit rounding rule.

### Order-book checksums

Kraken publishes a CRC32 over the top ten levels on each side of its book. Each price and quantity is written as it was sent, the decimal point and leading zeros are removed, asks are concatenated before bids, and the result is hashed. A stored mantissa at the venue's own exponent is exactly that digit string, so the checksum can be reproduced without formatting.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Kraken WebSocket v2: book checksum guide](https://docs.kraken.com/api/docs/guides/spot-ws-book-v2) | Primary | 2026-11-15 |
| [Binance SBE stream schema](https://developers.binance.com/en/docs/products/spot/sbe/schemas/stream_1_0.xml) | Primary | 2026-11-15 |
| [Bybit SBE basic information](https://bybit-exchange.github.io/docs/zh-TW/v5/sbe/sbe-basic-info) | Primary | 2026-11-15 |
| [NautilusTrader: `fixed.rs` (precision constants and raw types)](https://nautilustrader.io/docs/rust-api-latest/src/nautilus_model/types/fixed.rs.html) | Primary | 2026-11-15 |
| [NautilusTrader: overview (high-precision build in Python packages)](https://nautilustrader.io/docs/latest/concepts/overview/) | Primary | 2026-11-15 |
| [Databento: instrument definitions (fixed-point prices)](https://databento.com/docs/schemas-and-data-formats/instrument-definitions) | Primary | 2026-11-07 |
| [NautilusTrader adapter survey, section 1](../nautilus-adapter-survey.md) | Secondary | 2026-11-07 |
