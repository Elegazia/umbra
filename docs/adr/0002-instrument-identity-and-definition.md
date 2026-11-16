# ADR 0002: Instrument identity and definition schema

- Status: Accepted
- Date: 2026-11-15
- Deciders: Project owner
- Related: [ADR 0001](0001-brokers-venues-and-data-sources.md), ADR 0004, [ADR 0005](0005-numeric-representation.md), [ADR 0006](0006-market-data-records-and-timestamps.md)

## Context

Every record Umbra stores (trades, order-book updates, candles) refers to an instrument, and every check on a record depends on that instrument's definition: whether a price is on the tick grid, whether a quantity is a valid lot, whether the instrument was trading at the time. This ADR decides how instruments are identified and how their definitions are stored.

The same data serves two purposes. For research, years of recorded data are queried and backtested, and mistakes in identity or definitions corrupt results without any visible error. For trading, live data drives signals and orders, which have to use the venue's current symbol and rules.

The venues in scope ([ADR 0001](0001-brokers-venues-and-data-sources.md)) disagree on almost every aspect of an instrument. The [instrument definition field survey](../instrument-definition-field-survey.md) has the details. The points that matter here are these.

Names are neither stable nor unique. Kraken calls Bitcoin `XXBT` in REST and `XBT` on WebSocket, while Binance calls it `BTC`. IBKR's `symbol` (`ES`, for example) matches one contract per expiry, and only `conId` is unique. Tickers are renamed (Nasdaq `FB` became `META` in June 2022) and reused, and Databento warns that some publishers remap numeric instrument ids daily.

Definitions change. Tick sizes, minimum sizes and trading status change over time, and Kraken removed its fee fields on 8 September 2026.

Tick sizes are not always a single number. Kraken and Binance publish one tick size per pair, but equity markets often set the tick by price band. SGX Rule 8.3.3 sets S$0.001 below S$0.20, S$0.005 from S$0.20 to S$0.995 and S$0.01 from S$1.00, with a different table for structured warrants on the same exchange. IBKR publishes such bands as market rules, per exchange.

Venue precision fields are unreliable. Binance reports a `baseAssetPrecision` of 8 for BTCUSDT, while the actual quantity step is 0.00001.

This has to be settled before ADR 0006 (record schemas), because every record carries the instrument key defined here.

## Decision drivers

Hard constraints:

- D1. Point-in-time correctness. For any record, the instrument it belongs to and the rules that applied to it are those in force at the record's event time, not at query time.
- D2. Coverage of the venues in ADR 0001, including IBKR contracts that share a symbol and price-banded tick sizes.
- D3. No information loss. Fields that are not modelled today can be recovered from stored data, so definitions can be reprocessed later.

Preferences, in order:

- D4. One identifier across research and trading, so a strategy uses the same key live as in its backtest.
- D5. Self-describing data. Stored definitions can be interpreted from the data alone, without code that has to be kept in step with it.
- D6. No code changes for routine venue changes. A venue changing a tick table or listing an instrument changes data, not source code.
- D7. Negligible cost on the hot path compared with parsing.
- D8. Simplicity: few tables and one pattern for data that varies over time.

## Considered options

The decision has three parts: the key, how definitions change over time, and how price increments are represented.

### 1. Instrument key

1A. Natural key `(venue, venue_symbol)`. The venue's own identifier is the key.

- Good, because it needs no lookup and no extra table (D7, D8).
- Bad, because a rename splits one instrument's history across two keys, and a reused symbol merges two instruments under one key, without any error (fails D1).
- Bad, because it does not work for IBKR unless `conId` replaces the symbol (D2), and a rename between backtest and deployment breaks the link between them (D4).

1B. Surrogate key with an effective-dated mapping. Umbra assigns an integer `instrument_id`, and a mapping table records which venue identifiers referred to it and when.

- Good, because the mapping's validity ranges resolve each identifier as of the event date, so renames and reuse are handled (D1).
- Good, because each IBKR contract gets its own id, with `conId` stored in the mapping (D2), and the same id is used live and in backtests (D4).
- Good, because the lookup happens once per subscription and costs one dictionary access on the hot path (D7).
- Bad, because it needs an extra table and an as-of lookup at ingest (D8).

1C. Structured readable key, like NautilusTrader's `InstrumentId` (`BTCUSDT.BINANCE`). A string built from a normalised symbol and the venue.

- Good, because it is readable in logs and needs only a string builder (D8).
- Bad, because the key is still derived from a name, so it fails D1 and D4 in the same way as 1A, and it covers IBKR (D2) only if the expiry is encoded in the string.

Databento and QuantConnect LEAN both keep a permanent internal identifier with date-ranged symbol mappings, for exactly the rename and reuse cases ([Databento symbology](https://databento.com/docs/standards-and-conventions/symbology); [LEAN security identifiers](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/security-identifiers)). Systems that only trade live, such as CCXT and vn.py, key on current names, which is enough when no history is kept.

### 2. Definitions over time

2A. Latest only. The definition row is overwritten when anything changes.

- Good, because it is the simplest (D8).
- Bad, because historical records are validated against today's rules (fails D1), and superseded definitions are lost (fails D3).

2B. Effective-dated rows. Each definition row has `valid_from` and `valid_to`, and a change closes the current row and opens a new one.

- Good, because each record can be validated against the definition in force at its event time (D1), and no definition is lost (D3).
- Good, because it follows the same pattern as the mapping in 1B (D8), and a row is written only when something changes.
- Bad, because it needs change detection, and its precision is limited by how often definitions are fetched.

2C. Daily snapshots. The full definition of every instrument is stored once a day.

- Good, because it is simple to write and keeps history to the day (D1, D3).
- Bad, because storage grows every day whether or not anything changed, and a change during the day is not captured.

Databento publishes definitions as a stream of changes, each with an action (added, modified, deleted) and an event timestamp, and describes point-in-time definitions as the way to avoid look-ahead and survivorship bias ([Databento instrument definitions](https://databento.com/docs/schemas-and-data-formats/instrument-definitions)). NautilusTrader stamps each definition with `ts_event` and `ts_init` ([NautilusTrader instruments](https://nautilustrader.io/docs/latest/concepts/instruments/)).

### 3. Price increments

3A. A single scalar `tick_size`.

- Good, because validation is one operation (D7) and the value is self-describing (D5).
- Bad, because it cannot represent banded markets (fails D2). On SGX, a price of S$0.995 is valid (199 × 0.005) but fails a check against 0.01.

3B. A list of price bands `[(low, increment), …]`, ordered by `low`. The first band starts at 0, and each band applies from its `low` (inclusive) up to the next band's `low` (exclusive). An instrument with one tick size has one band.

- Good, because it represents every venue in scope and matches IBKR's market rules (`lowEdge`, `increment`) directly (D2).
- Good, because the bands are stored with the data and come from the venue, so neither reading old data nor handling a changed tick table needs code changes (D5, D6).
- Bad, because validation needs a binary search over the bands, at most about ten (D7).

3C. A named tick scheme: a string referring to a registry of schemes defined in code, as in NautilusTrader's `tick_scheme`.

- Good, because instruments that share a table share one definition.
- Bad, because the data holds only a name, and reading it needs code that defines the scheme with the bands that applied at the time (fails D5).
- Bad, because a new or changed table is a code change (fails D6).

Of five open-source systems surveyed, only NautilusTrader supports variable tick sizes, through named schemes. Its documentation notes that custom schemes must be registered before saved definitions can be loaded ([open-source instrument model survey, section 2.4](../open-source-instrument-model-survey.md)). Its IBKR adapter nevertheless uses a single `min_tick` ([NautilusTrader adapter survey, section 5](../nautilus-adapter-survey.md)). IBKR documents that `minTick` is only the smallest increment across all exchanges and prices, and that the full structure comes from `reqMarketRule` ([IBKR minimum price increment](https://www.interactivebrokers.com/docs/tws-api/doc/orders/minimum-price-increment)).

## Decision

Chosen: 1B, 2B and 3B. Instruments are identified by a surrogate `instrument_id` with an effective-dated mapping from venue identifiers; definitions are stored as effective-dated rows; and price increments are stored as a list of price bands.

Umbra serves both research and live trading. For research, an instrument's history has to stay correct when a venue renames or reuses a symbol, and only a surrogate id with dated mappings guarantees that. For trading, the surrogate id costs nothing in speed once a subscription is set up, and a strategy uses the same id live as in its backtest. 1B is the only key that meets D1 and D2 together.

Checking historical data has to use the rules that applied on the day. Effective-dated rows provide this, follow the same pattern as the mapping table, and add a row only when something actually changes. 2B meets D1 and D3 without the storage growth of 2C.

Price bands are flexible enough for every venue in scope, and when an exchange changes its tick table, that should change data, not code. The speed cost is the same as a scalar in practice. 3B is the only representation that meets D2, D5 and D6 together. It is also the format IBKR already publishes, and an instrument with a single tick size is simply a list with one band.

### Schema

All timestamps are UTC. All validity ranges are half-open, `[valid_from, valid_to)`, with `valid_to` null for the current row. Numeric fields use the exact type decided in ADR 0005. Until then they are parsed from the venue's strings into `Decimal`, never through `float`.

`instrument_mapping` records which venue identifiers refer to which instrument, and when.

| Field | Type | Notes |
|---|---|---|
| `venue` | string | Umbra's venue code |
| `id_kind` | string | Which venue identifier this is: `symbol`, `ws_symbol`, `con_id`, and so on. Kraken has two names per pair. |
| `venue_value` | string | The identifier as the venue sends it |
| `instrument_id` | int64 | |
| `valid_from`, `valid_to` | timestamp | No two rows for the same `(venue, id_kind, venue_value)` overlap |

`instrument_definition` holds one row per instrument for each period in which its definition did not change.

| Field | Type | Notes |
|---|---|---|
| `instrument_id` | int64 | |
| `valid_from`, `valid_to` | timestamp | |
| `venue` | string | |
| `canonical_symbol` | string | Readable name for logs and paths, such as `BTC/USDT`; not a key |
| `asset_class` | enum | `crypto`, `equity`, `fund`, `future`, `option`, `fx` |
| `instrument_type` | enum | `spot`, `perpetual`, `future`, `option`, `stock`, `etf`, and so on |
| `base_asset`, `quote_asset` | string | Normalised asset codes; `base_asset` is null where it does not apply |
| `status` | enum | `trading`, `halted`, `cancel_only`, `post_only`, `reduce_only`, `delisted`, `unknown` |
| `price_increments` | list of `(low, increment)` | Sorted by `low`; the first `low` is 0 |
| `quantity_step` | exact decimal | The venue's step, stored as given |
| `min_quantity`, `min_notional` | exact decimal, nullable | |
| `multiplier`, `expiry`, `underlying_instrument_id`, `strike`, `option_right` | nullable | Null for spot instruments |
| `fetched_at` | timestamp | When the source response was obtained |
| `source` | string | The endpoint or file the definition came from |
| `raw` | string (JSON) | The venue's full response for this instrument |

`asset_alias` maps `(venue, venue_code)` to `asset_code`, for example Kraken's `XXBT` and `XBT` to `BTC`.

### Rules

1. Ingest looks up `instrument_id` using each record's event timestamp. If no mapping row matches, the record is quarantined. It is never given the current id instead.
2. Mappings and definitions are append-only. Rows are closed by setting `valid_to` and new rows are added; existing rows are never edited, so reprocessing gives the same result.
3. A venue identifier with no mapping is treated as a new instrument. Linking a renamed identifier to an existing id is done with an explicit mapping entry, because venues do not reliably say whether a new symbol is a rename. Where IBKR's `conId` is available it is used for the link, since it survives ticker changes.
4. Definition changes are detected by comparison. On each fetch, the normalised definition is compared with the open row. If any field other than `fetched_at`, `source` and `raw` differs, the open row is closed at the fetch time and a new one is opened.
5. Increments are the primary fact, and decimal places are derived from them. Venue precision fields are not used. Where a protocol needs a venue-specific decimal count, such as Kraken's `pair_decimals` for order-book checksums, that venue's adapter reads it from `raw`.
6. Asset codes are normalised through `asset_alias`, never by string rules; stripping a leading `X` would turn Tezos `XTZ` into `TZ`. Unknown codes pass through unchanged and are logged.
7. Instruments that are not trading are stored with their status rather than rejected, so halted and delisted instruments keep their history.
8. Fees are not part of the definition. They depend on the account and change independently, and are modelled together with execution.
9. A field the venue does not provide is stored as null. Defaults are never guessed.
10. If the current venue identifier for an `instrument_id` cannot be resolved, orders for that instrument are refused.

## Consequences

Backtests read each instrument's history under one key across renames, and validate each record against the rules of its day. Adding a venue is adapter work (parsing into this schema and adding `asset_alias` rows) rather than a schema change. Definitions can be reprocessed from `raw` when new fields are modelled.

Every ingest path needs an as-of lookup, and every definition fetch needs change detection. Renames have to be linked explicitly; until a rename is linked, it appears as a new instrument. Readable output needs a join from `instrument_id` to `instrument_definition`.

A change made between two fetches is dated at the later fetch, so records in that window may be validated against the old rules. Records that fail validation are quarantined rather than dropped, so they can be checked again, and venues that publish change feeds are used where available.

Overlapping or gapped validity ranges would make lookups ambiguous. A check that each table's ranges neither overlap nor leave gaps runs in the test suite and after every write.

Keeping `raw` costs some storage, but definitions are small and change rarely, so this is negligible next to market data.

Follow-up work:

- ADR 0005 decides the exact numeric type for increments, prices and quantities. A modulo check in binary floating point misclassifies valid prices: `65000.3 % 0.1` evaluates to `0.0999…`, not 0.
- ADR 0006 defines the market-data records that carry `instrument_id`.
- ADR 0004 decides whether storage paths use `instrument_id` or `canonical_symbol`.
- The Coinbase product endpoint is surveyed before the Coinbase adapter is written.

### Revisit when

- a venue in scope varies the quantity step by size (add quantity bands, as for prices);
- more than a handful of renames a month need manual linking (use an external symbology source, such as Databento symbology or OpenFIGI, to link them);
- records quarantined because a definition changed between fetches exceed 0.1% of a day's records for any venue (fetch more often, or subscribe to the venue's change feed);
- a typed model with one class per instrument type is needed, for example in a compiled core (add it as a layer over this storage schema, and supersede this ADR only if the storage schema itself changes).

## Background

### Natural and surrogate keys

A natural key is an identifier that already exists in the domain, such as a ticker. A surrogate key is generated by the system and has no meaning outside it. The usual guidance in database design is to use a surrogate primary key when the natural candidate is not guaranteed to be unique, stable and always present. Market identifiers are not stable, because of renames, reuse and daily remapping, and for derivatives they are not unique either, because one root symbol covers many contracts.

### Effective dating and point-in-time data

An effective-dated table, called a type 2 slowly changing dimension in data warehousing, keeps every version of a row together with the period in which it was true. A query "as of time t" selects the row whose range contains t. Half-open ranges `[from, to)` make sure that consecutive versions neither overlap nor leave gaps. Backtests need this because interpreting past data with today's facts introduces look-ahead bias (using information that was not available at the time) and survivorship bias (only instruments that still exist appear in the history).

### Tick sizes and price bands

The tick size is the smallest price increment an order book accepts. Many equity markets set it by price band, so that one tick is a similar fraction of the price for cheap and expensive securities; SGX's table is one example. A price p is valid in the band (low, increment) when p is an exact multiple of the increment, which requires exact decimal arithmetic.

### Keeping the raw response

Keeping the venue's original response next to the parsed fields is common practice (CCXT `info`, NautilusTrader `info`, vn.py `extra`). The parsed fields serve queries. The raw response lets definitions be parsed again when the model grows, without fetching history that the venue may no longer provide.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Instrument definition field survey (Kraken, Binance, IBKR)](../instrument-definition-field-survey.md) | Primary (venue responses and API references) | 2026-11-07 |
| [Instrument models in open-source trading systems](../open-source-instrument-model-survey.md) | Secondary (project documentation) | 2026-11-07 |
| [NautilusTrader adapters: how venue fields become instruments](../nautilus-adapter-survey.md) | Secondary (source code) | 2026-11-07 |
| [SGX Rulebook, Rule 8.3.3 Minimum Bid Size (version effective 8 Oct 2018)](https://rulebook.sgx.com/node/1104) | Primary | 2026-11-15 |
| [IBKR TWS API: minimum price increment and market rules](https://www.interactivebrokers.com/docs/tws-api/doc/orders/minimum-price-increment) | Primary | 2026-11-15 |
| [Databento: symbology](https://databento.com/docs/standards-and-conventions/symbology) | Primary | 2026-11-07 |
| [Databento: instrument definitions](https://databento.com/docs/schemas-and-data-formats/instrument-definitions) | Primary | 2026-11-07 |
| [NautilusTrader: instruments](https://nautilustrader.io/docs/latest/concepts/instruments/) | Primary | 2026-11-07 |
| [QuantConnect LEAN: security identifiers](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/security-identifiers) | Primary | 2026-11-07 |
| [CCXT Manual](https://raw.githubusercontent.com/ccxt/ccxt/master/wiki/Manual.md) | Primary | 2026-11-07 |
