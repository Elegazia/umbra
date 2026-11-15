# ADR 0002: Instrument identity and definition schema

- **Status:** Accepted
- **Date:** 2026-11-15
- **Deciders:** Project owner
- **Related:** [ADR 0001](0001-brokers-venues-and-data-sources.md) (venues in scope); ADR 0004 (storage format and query engine, planned); ADR 0005 (numeric representation, planned); ADR 0006 (market-data record schemas, planned)

## Context

Every record Umbra stores (trades, order-book updates, candles) refers to an instrument, and every check on that record (is this price on the tick grid, is this quantity a valid lot, was the instrument trading) depends on the instrument's definition. This ADR decides how an instrument is identified and how its definition is stored.

Umbra serves two purposes with the same data:

1. **Research:** years of recorded data are queried and backtested. Errors in identity or definitions corrupt results silently.
2. **Trading:** live data drives signals and orders, which must use the venue's current symbol and rules.

The venues in scope ([ADR 0001](0001-brokers-venues-and-data-sources.md)) disagree on almost every aspect of an instrument. The [instrument definition field survey](../instrument-definition-field-survey.md) records the details; the facts that shape this decision are:

- **Names are not stable or unique.** Kraken spells Bitcoin `XXBT` (REST) and `XBT` (WebSocket), Binance spells it `BTC`. IBKR's `symbol` (`ES`) matches one contract per expiry; only `conId` is unique. Tickers are renamed (Nasdaq `FB` became `META` in June 2022) and reused, and Databento warns that some publishers remap numeric instrument ids daily.
- **Definitions change.** Tick sizes, minimums and status change over time; Kraken removed its fee fields on 8 September 2026.
- **Tick sizes are not always one number.** Kraken and Binance publish one tick size per pair. Equity markets often set the tick by price band: SGX Rule 8.3.3 sets S$0.001 below S$0.20, S$0.005 from S$0.20 to S$0.995, and S$0.01 from S$1.00, with a different table for structured warrants on the same exchange. IBKR publishes these bands as *market rules*, per exchange.
- **Venue "precision" fields mislead.** Binance reports `baseAssetPrecision` 8 for BTCUSDT while the real quantity step is 0.00001.

The decision is needed before ADR 0006 (record schemas), because every record carries the instrument key defined here.

## Decision drivers

Hard constraints:

1. **D1 (hard): Point-in-time correctness.** For any record, the instrument it belongs to and the rules that applied to it are those in force at the record's event time, not at query time.
2. **D2 (hard): Coverage of the venues in ADR 0001**, including IBKR contracts that share a symbol and price-banded tick sizes.
3. **D3 (hard): No information loss.** Fields not modelled today must remain recoverable from stored data, so that definitions can be reprocessed later.

Preferences, ranked:

4. **D4: One identifier across research and trading.** A strategy uses the same instrument key live as in its backtest.
5. **D5: Self-describing data.** Stored definitions can be interpreted from the data alone, without code that must be kept in sync.
6. **D6: No per-change code edits.** A venue changing a tick table or listing an instrument updates data, not source code.
7. **D7: Negligible hot-path cost.** Per-message overhead is small relative to parsing.
8. **D8: Simplicity.** Few tables, one pattern for time-varying data.

## Options considered

The decision has three parts that are evaluated separately: the key, how definitions vary over time, and how price increments are represented.

### Part 1: Instrument key

**1A. Natural key `(venue, venue_symbol)`.** The venue's own identifier is the key.

**1B. Surrogate key with an effective-dated mapping.** Umbra assigns an integer `instrument_id`. A mapping table records which venue identifiers referred to it, and when.

**1C. Structured readable key** (as NautilusTrader's `InstrumentId`, e.g. `BTCUSDT.BINANCE`). A string built from a normalised symbol and the venue.

| Driver | 1A natural | 1B surrogate + mapping | 1C structured |
|---|---|---|---|
| D1 point-in-time | Fails: a rename splits one instrument's history across two keys; a reused symbol merges two instruments under one key, without error | Holds: the mapping's validity ranges resolve each identifier as of the event date | Fails, as 1A: the key is derived from a name |
| D2 coverage | Fails for IBKR unless `conId` replaces the symbol | Holds: each contract has its own id; the mapping stores `conId` | Holds only if expiry is encoded in the string |
| D4 research = live | Breaks if the venue renames between backtest and deployment | Holds | Breaks, as 1A |
| D7 hot path | No lookup | One dictionary lookup per subscription, resolved once at subscribe time | No lookup |
| D8 simplicity | Simplest | One extra table and an as-of lookup at ingest | A string builder |

Precedents: Databento and QuantConnect LEAN keep a permanent internal identifier and date-ranged symbol mappings for exactly the rename and reuse cases ([Databento symbology](https://databento.com/docs/standards-and-conventions/symbology); [LEAN security identifiers](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/security-identifiers)). Live-only systems (CCXT, vn.py) key on current names, which is sufficient when no history is kept.

### Part 2: Definitions over time

**2A. Latest only.** The definition row is overwritten when anything changes.

**2B. Effective-dated rows.** Each definition row has `valid_from` and `valid_to`. A change closes the current row and opens a new one.

**2C. Daily snapshots.** The full definition of every instrument is stored once per day.

| Driver | 2A latest | 2B effective-dated | 2C snapshots |
|---|---|---|---|
| D1 point-in-time | Fails: historical records are validated against today's rules | Holds, to the precision of the fetch interval | Holds, to the day |
| D3 no loss | Fails: superseded definitions are lost | Holds | Holds |
| D8 simplicity | Simplest | Same pattern as the Part 1 mapping; needs change detection | Simple to write; storage grows daily whether or not anything changed |

Precedents: Databento publishes definitions as a stream of changes with an action (added, modified, deleted) and an event timestamp, and describes point-in-time definitions as the way to avoid look-ahead and survivorship bias ([Databento instrument definitions](https://databento.com/docs/schemas-and-data-formats/instrument-definitions)). NautilusTrader stamps each definition with `ts_event` and `ts_init` ([NautilusTrader instruments](https://nautilustrader.io/docs/latest/concepts/instruments/)).

### Part 3: Price increments

**3A. One scalar `tick_size`.**

**3B. A list of price bands** `[(low, increment), …]`, ordered by `low`, where the first band starts at 0 and each band applies from its `low` (inclusive) to the next band's `low` (exclusive). A single-tick instrument has one band.

**3C. A named tick scheme**, a string referring to a registry of schemes defined in code (NautilusTrader's `tick_scheme`).

| Driver | 3A scalar | 3B band list | 3C named scheme |
|---|---|---|---|
| D2 coverage | Fails for banded markets: SGX price S$0.995 is valid (199 × 0.005) but fails a 0.01 check | Holds; matches IBKR market rules (`lowEdge`, `increment`) directly | Holds |
| D5 self-describing | Holds | Holds | Fails: data holds a name; reading it needs the code that defines the scheme, with the bands that applied at the time |
| D6 no code edits | Holds | Holds: bands come from the venue's data | Fails: a new or changed table is a code change |
| D7 hot path | One operation | Binary search over at most about ten bands | One registry lookup, then as 3B |

Precedents: of five open-source systems surveyed, only NautilusTrader supports variable tick sizes, through named schemes; its documentation notes that custom schemes must be registered before saved definitions can be loaded ([open-source instrument model survey §2.4](../open-source-instrument-model-survey.md)). Its IBKR adapter nevertheless uses a single `min_tick` ([NautilusTrader adapter survey §5](../nautilus-adapter-survey.md)). IBKR documents that `minTick` is only the smallest increment across all exchanges and prices, and that the full structure comes from `reqMarketRule` ([IBKR minimum price increment](https://www.interactivebrokers.com/docs/tws-api/doc/orders/minimum-price-increment)).

## Decision

**1B, 2B and 3B:** a surrogate `instrument_id` with an effective-dated mapping from venue identifiers; definitions stored as effective-dated rows; price increments stored as a list of price bands.

### Schema

All timestamps are UTC. All validity ranges are half-open, `[valid_from, valid_to)`, with `valid_to` null for the current row. Numeric fields use the exact type decided in ADR 0005; until then they are parsed from the venue's strings into `Decimal`, never through `float`.

**`instrument_mapping`**: which venue identifiers refer to which instrument, and when.

| Field | Type | Notes |
|---|---|---|
| `venue` | string | Umbra's venue code |
| `id_kind` | string | Which venue identifier: `symbol`, `ws_symbol`, `con_id`, … (Kraken has two names per pair) |
| `venue_value` | string | The identifier as the venue sends it |
| `instrument_id` | int64 | |
| `valid_from`, `valid_to` | timestamp | No two rows for the same `(venue, id_kind, venue_value)` overlap |

**`instrument_definition`**: one row per instrument per period in which its definition was unchanged.

| Field | Type | Notes |
|---|---|---|
| `instrument_id` | int64 | |
| `valid_from`, `valid_to` | timestamp | |
| `venue` | string | |
| `canonical_symbol` | string | Readable name for logs and paths (e.g. `BTC/USDT`); not a key |
| `asset_class` | enum | `crypto`, `equity`, `fund`, `future`, `option`, `fx` |
| `instrument_type` | enum | `spot`, `perpetual`, `future`, `option`, `stock`, `etf`, … |
| `base_asset`, `quote_asset` | string | Normalised asset codes; `base_asset` null where not applicable |
| `status` | enum | `trading`, `halted`, `cancel_only`, `post_only`, `reduce_only`, `delisted`, `unknown` |
| `price_increments` | list of `(low, increment)` | Sorted by `low`; first `low` is 0 |
| `quantity_step` | exact decimal | The venue's step, stored as given |
| `min_quantity`, `min_notional` | exact decimal, nullable | |
| `multiplier`, `expiry`, `underlying_instrument_id`, `strike`, `option_right` | nullable | Null for spot instruments |
| `fetched_at` | timestamp | When the source response was obtained |
| `source` | string | Endpoint or file the definition came from |
| `raw` | string (JSON) | The venue's full response for this instrument |

**`asset_alias`**: `(venue, venue_code) → asset_code` (Kraken `XXBT` and `XBT` → `BTC`).

### Rules

1. **Ids are resolved as of event time.** Ingest looks up `instrument_id` using each record's event timestamp. If no mapping row matches, the record is quarantined; it is never assigned the current id.
2. **Mappings and definitions are append-only.** Rows are closed (by setting `valid_to`) and added, never edited, so reprocessing gives the same result.
3. **New identifiers get new ids.** A venue identifier with no mapping is a new instrument. Linking a renamed identifier to an existing id is an explicit mapping entry, because venues do not reliably announce whether a new symbol is a rename. IBKR `conId`, which survives ticker changes, is used for that link where available.
4. **Definition changes are detected by comparison.** On each fetch, the normalised definition is compared with the open row; if any field other than `fetched_at`, `source` and `raw` differs, the open row is closed at the fetch time and a new row is opened.
5. **Increments are primary; precision is derived.** Price and quantity decimal places are computed from the increments, never read from venue precision fields. Venue-specific decimal counts that are needed for a protocol (Kraken's `pair_decimals` for order-book checksums) are read from `raw` by that venue's adapter.
6. **Asset codes are normalised through `asset_alias`**, never by string rules (stripping a leading `X` would turn Tezos `XTZ` into `TZ`). Unknown codes pass through unchanged and are logged.
7. **Non-trading instruments are stored** with their status, not rejected, so halted and delisted instruments keep their history.
8. **Fees are not part of the definition.** They depend on the account and change independently; they are modelled with execution.
9. **No invented values.** A field the venue does not provide is null, never a guessed default.
10. **Trading fails closed.** If the current venue identifier for an `instrument_id` cannot be resolved, orders for that instrument are refused.

### Rationale

1B is the only key that satisfies D1 and D2 together, and it adds D4; its cost is a mapping table and an as-of lookup at ingest, both off the per-message path (D7). 2B satisfies D1 and D3 with the same effective-dated pattern as the mapping (D8), storing a row only when something changes. 3B is the only increment representation that satisfies D2, D5 and D6 together; it is the format IBKR already publishes, and a single-tick instrument is a one-band list, so the common case costs nothing extra.

**Owner's rationale:**

- *Key:* Umbra serves both research and live trading. For research, an instrument's history must stay correct when a venue renames or reuses a symbol, which only a surrogate id with dated mappings guarantees. For trading, it costs nothing in speed after subscription, and a strategy uses the same id live as in its backtest.
- *Definitions over time:* checking historical data has to use the rules that applied on that day. Dated rows give that, follow the same pattern as the mapping table, and only add a row when something actually changes.
- *Price increments:* bands are flexible enough for every venue, and an exchange changing its tick table should change data, not code. The cost in speed is the same.

## Consequences

- **Easier:**
  - Backtests read history under one key per instrument across renames, and validate each record against the rules of its day.
  - Adding a venue is adapter work (parsing into this schema plus `asset_alias` rows), not a schema change.
  - Definitions can be reprocessed from `raw` when new fields are modelled.
- **Harder:**
  - Every ingest path needs an as-of lookup, and every definition fetch needs change detection.
  - Renames must be linked explicitly; an unlinked rename appears as a new instrument until it is.
  - Readable output needs a join from `instrument_id` to `instrument_definition`.
- **Risks and mitigations:**
  - *Change timing is limited by fetch frequency:* a change between fetches is dated at the later fetch. Records in that window may be validated against old rules; validation failures are quarantined, not dropped, so they can be re-checked. Venues with change feeds are used where available.
  - *Overlapping or gapped validity ranges* would make lookups ambiguous. A uniqueness check on each table's ranges runs in the test suite and after every write.
  - *`raw` grows storage.* Definitions are small and change rarely; this is negligible next to market data.
- **Follow-up work:**
  - ADR 0005 fixes the exact numeric type for increments, prices and quantities. A modulo check in binary floating point misclassifies valid prices (`65000.3 % 0.1` evaluates to `0.0999…`, not 0).
  - ADR 0006 defines the market-data records that carry `instrument_id`.
  - ADR 0004 decides whether storage paths use `instrument_id` or `canonical_symbol`.
  - Survey the Coinbase product endpoint before the Coinbase adapter is written.

### Revisit triggers

- A venue in scope varies the quantity step by size → add quantity bands, as for prices.
- More than a handful of renames per month need manual linking → adopt an external symbology source (for example Databento symbology or OpenFIGI) to link them.
- Quarantined records caused by definition changes between fetches exceed 0.1% of a day's records for any venue → fetch more often or subscribe to the venue's change feed.
- A typed, class-per-instrument-type model is needed (for example in a compiled core) → add it as a layer over this storage schema; supersede this ADR only if the storage schema itself changes.

## Background

### Natural and surrogate keys

A **natural key** is an identifier that already exists in the domain, such as a ticker. A **surrogate key** is generated by the system and has no meaning outside it. The standard guidance in database design is to use a surrogate primary key when the natural candidate is not guaranteed to be unique, stable and always present. Market identifiers fail stability (renames, reuse, daily remapping) and, for derivatives, uniqueness (one root symbol for many contracts).

### Effective dating and point-in-time data

An effective-dated table (in data-warehouse terms, a *type 2 slowly changing dimension*) keeps every version of a row with the period during which it was true. A query "as of time *t*" selects the row whose range contains *t*. Half-open ranges `[from, to)` ensure that consecutive versions neither overlap nor leave gaps. Backtests need this because using today's facts to interpret past data introduces **look-ahead bias** (using information not available at the time) and **survivorship bias** (only instruments that still exist appear in history).

### Tick sizes and price bands

The tick size is the minimum price increment an order book accepts. Many equity markets set it by price band so that one tick is a similar fraction of price for cheap and expensive securities; SGX's table is an example. A price *p* is valid in band *(low, increment)* when *p* is an exact multiple of the increment, which requires exact decimal arithmetic.

### The raw escape hatch

Keeping the venue's original response alongside parsed fields is common practice (CCXT `info`, NautilusTrader `info`, vn.py `extra`). Parsed fields serve queries; the raw payload lets definitions be reparsed when the model grows, without re-fetching history that the venue may no longer serve.

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
