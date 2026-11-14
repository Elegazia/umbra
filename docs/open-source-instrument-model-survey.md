# Instrument models in open-source trading systems

- **Date:** 2026-11-07
- **Purpose:** Input to [ADR 0002](0002-Record-Schema-and-DataSource-Interface). How established projects decided to model instruments, and why those decisions matter for Umbra. Companion to [instrument-definition-field-survey.md](instrument-definition-field-survey.md), which covers what Kraken, Binance and IBKR actually provide.
- **Status:** Survey, not a decision.

## How this was checked

Documentation and a few source files were read through a fetch tool that summarises pages. No code was run or cloned.

| Project | What was read | Gaps |
|---|---|---|
| [CCXT](https://github.com/ccxt/ccxt) | Manual, "Market Structure" section | Numeric type of precision and limits not stated in the section |
| [NautilusTrader](https://github.com/nautechsystems/nautilus_trader) | Instruments and value-types docs, `currency_pair.rs` | Other instrument classes not read. Fixed-point detail is from the docs, not the Rust source. |
| [Databento](https://databento.com/docs) | Instrument-definition schema and symbology docs | How `instrument_id` remapping works internally is only partly described |
| [QuantConnect LEAN](https://github.com/QuantConnect/Lean) | Security-identifier docs | Delisting handling not described |
| [vn.py](https://github.com/vnpy/vnpy) | `ContractData` in `object.py` | `Product` and `Exchange` enum members not shown |

Not covered: Hummingbot, Zipline, Backtrader, Freqtrade, QuantLib, ib_async. Nautilus is the closest architectural match to Umbra (Rust core, Python layer, event-driven), so it gets the most weight.

---

## 1. What each project does

| | CCXT | NautilusTrader | Databento | LEAN | vn.py |
|---|---|---|---|---|---|
| Role | Exchange API wrapper | Trading engine | Data vendor | Trading engine | Trading platform |
| Shape | One dict per market, boolean flags for the type | A class per instrument type (about 19) | One flat record, `instrument_class` field | `Symbol` object wrapping a `SecurityIdentifier` | One `ContractData` class, optional option fields |
| Venue-independent key | `symbol`, e.g. `BTC/USDT` | `InstrumentId` = `symbol.venue` | Numeric `instrument_id` | `SecurityIdentifier` (encoded) | `vt_symbol` = `symbol.exchange` |
| Venue's own name kept | `id` | `raw_symbol` | `raw_symbol` | `value` (the ticker) | `symbol` |
| Price grid | `precision` (tick size mode) plus `limits` | `price_precision` and `price_increment` | `min_price_increment` | not read | `pricetick` (float) |
| Quantity grid | `precision.amount` | `size_precision` and `size_increment` | `min_lot_size` and variants | not read | `min_volume`, `max_volume` |
| Raw escape hatch | `info` | `info` | all fields in schema | not read | `extra` |
| Time fields | none | `ts_event`, `ts_init` | `ts_event`, `ts_recv` | date inside the ID | none |

## 2. Decisions and what they imply

### 2.1 Identity: three keys, not one

Every project separates at least two names for an instrument:

- **The venue's own name** (CCXT `id`, Nautilus and Databento `raw_symbol`, vn.py `symbol`). Needed to talk to the venue.
- **A name the system owns** (CCXT `symbol`, Nautilus `InstrumentId`, vn.py `vt_symbol`). Needed to refer to it internally and across venues.

CCXT's manual states the reason: venue ids "are unique per exchange and can't be used across exchanges". Nautilus builds the internal id as `symbol.venue` (e.g. `ETHUSDT-PERP.BINANCE`) and keeps `raw_symbol` separately.

Two projects add a warning about the *numeric* or *ticker* name:

- **Databento:** numeric `instrument_id`s are assigned by the publisher and "some publishers remap them daily", so one id can mean different instruments on different days. Their symbology endpoint returns mappings with a start and end date (`d0`, `d1`). Their rule: never retroactively reassign symbols in history, and treat each mapping as valid only within its date range.
- **LEAN:** a ticker is a label that changes with rebrands and mergers, and brokers reuse tickers. The `Symbol` stays fixed; the ticker is "not the same as the `Symbol`". Ticker lookup at a past date needs an explicit date (`mapping_resolve_date`).

**Implication for Umbra:** do not use a venue's numeric id or ticker as a permanent key. IBKR `conId` and Databento `instrument_id` are venue-assigned. An internal id that is yours, plus a dated mapping to venue ids, matches how Databento and LEAN avoid breakage.

### 2.2 Symbol strings: structured, but do not parse them

CCXT encodes the type in the symbol: `BTC/USDT` (spot), `BTC/USDT:USDT` (linear perpetual), `BTC/USDT:USDT-211225` (future), with options adding strike and `C`/`P`. Yet its manual advises using market *properties* instead of parsing the string. Nautilus likewise keeps `base_currency` and `quote_currency` as fields on the instrument.

**Implication:** a readable canonical symbol is useful for humans and logs, but `base`, `quote`, `type` and `expiry` should be real fields. This matters for the `USDT` vs `USD` distinction in ADR 0001.

### 2.3 Price and quantity grids: store both precision and increment

- **CCXT** has three `precisionMode`s: `TICK_SIZE`, `SIGNIFICANT_DIGITS` and `DECIMAL_PLACES`. The manual says `DECIMAL_PLACES` is deprecated, "CCXT no longer uses this mode anywhere", and that almost every exchange uses `TICK_SIZE`.
- **Nautilus** stores both `price_precision` (decimal places) and `price_increment` (a `Price`), and the constructor **rejects an instrument if they disagree**: the precision must equal the increment's precision, and the increment must be positive. The same applies to size.
- **Databento** stores `min_price_increment` as a fixed-point integer. A stored `10,000,000` at scale 1e-9 is a tick of 0.01. It also keeps a separate `display_factor` and fractional-display fields for venues that quote in fractions.
- **vn.py** stores `pricetick` as a plain float.

The lesson: decimal places and increment are two views of one fact. They can diverge, which is why Nautilus checks consistency at construction. Also notable: Nautilus's `RiskEngine` rejects orders with too many decimals, but it does **not** round them or check that a price is a multiple of the increment. The caller must do that.

**Implication:** store the increment as the primary fact and derive decimals, or store both and add a consistency check. This ties directly to the Binance finding that `baseAssetPrecision` (8) differs from the real step (0.00001).

### 2.4 Variable tick sizes: only Nautilus addresses it

IBKR's tick size depends on price band and exchange (see the field survey). Among the five projects, only Nautilus has a mechanism for this, an optional `tick_scheme` name that refers to a registry of tick schemes. The docs add that custom schemes "must be registered before loading saved definitions from a catalog", because loading validates the scheme name. The other projects use a single scalar (`tick_size`, `pricetick`, `min_price_increment`).

**Implication:** the scalar approach is enough for Kraken and Binance. Supporting IBKR equities means either a tick scheme (Nautilus) or a list of `(low_edge, increment)` pairs, as suggested in the field survey. Nautilus's registry trades flexibility against coupling between stored data and code.

### 2.5 Numeric representation

- **Nautilus** uses **scaled integers, not floats**: a global scale of 10^9, or 10^16 in a "high-precision mode". `Price` and `Quantity` each carry their own `precision` (decimal places) for display and serialisation. Equality depends only on the numeric value. `Quantity` is unsigned, and subtracting below zero raises. The docs justify the choice as "performant, deterministic calculations across different platforms".
- **Databento** uses `int64` fixed-point at 1e-9 for prices and related values.
- **vn.py** uses `float`.
- **CCXT:** the section read does not state the numeric type.

Two scale choices exist in practice (1e-9 and 1e-16) and both projects with fixed-point chose a global scale, not a per-instrument one. The cost is range: at 1e-9, `int64` caps values near 9.2 billion units.

**Implication:** this is the strongest precedent for the "Decimal in Python, scaled int at the C++ boundary" idea. Note that Nautilus and Databento both used scaled integers *in the data model*, not only at the boundary. Whether that is right for Umbra depends on the planned parser benchmark. Check that 1e-9 covers the smallest crypto quantities you intend to store.

### 2.6 Fees are not part of the instrument

- **Nautilus** keeps fee rates out of instruments entirely. Backtest commissions come from a fee model, and live commissions come from venue fills.
- **CCXT** puts `maker` and `taker` on the market but tells users to call `fetchTradingFees` for account-specific rates.

This agrees with the Kraken finding: Kraken deprecated `fees` and `fees_maker` on 8 September 2026 and now always returns `[]`. Fees depend on account tier and change often, so they behave like account data, not instrument data.

**Implication:** leave fees out of the instrument definition. Model them separately when execution begins.

### 2.7 Class per type vs one flat record

- **Nautilus:** about 19 classes (`CurrencyPair`, `Equity`, `FuturesContract`, `CryptoPerpetual`, `OptionContract`, ...). Each class only has fields that make sense for it. For `CurrencyPair`, required fields include precision, increments, multiplier and margins, while limits are optional.
- **Databento:** one flat record with a large field set and an `instrument_class` discriminator.
- **CCXT:** one dictionary with booleans (`spot`, `swap`, `future`, `option`) and fields that apply "only when `contract` is true".
- **vn.py:** one class where option fields are `None` for non-options.

Classes give type safety and prevent invalid combinations. A flat record is simpler to store, query and write to Parquet. Nautilus can afford classes because it is an engine that handles instruments as objects. Databento stores them in a columnar format and went flat.

**Implication:** for a first Parquet table with spot instruments, a flat record with nullable derivative fields is the simpler choice. A class-per-type model suits the C++ core later, where invalid states should be unrepresentable. The two can coexist if the flat record is the storage format.

### 2.8 Raw data and time

- **Raw escape hatch:** CCXT (`info`), Nautilus (`info`), and vn.py (`extra`) all keep the venue's unparsed data. Databento's schema carries a very large field set instead.
- **Time:** Nautilus stamps `ts_event` (when the definition applies) and `ts_init` (when your system created the object). Databento stamps `ts_event` and `ts_recv`, and adds `security_update_action` with values Added, Modified or Deleted, so a definition feed is a **time series of changes**. For some publishers it also provides a full snapshot of active definitions at UTC midnight on weekdays. Databento's docs say point-in-time definitions let you "avoid lookahead and survivorship bias". CCXT caches definitions per instance (`loadMarkets`) with no history, and vn.py keeps none.
- **Updates:** Nautilus lets strategies subscribe to instrument updates, and the data engine calls `on_instrument()`.
- **Loading:** Nautilus requires an instrument to be in a central cache before an order can be submitted, and loads it through a per-venue `InstrumentProvider`.

**Implication:** the projects that care about backtests (Databento, Nautilus, LEAN) all treat definitions as time-stamped. CCXT and vn.py, which are live-trading tools, do not. Since Umbra's scope includes backtesting, versioned definitions or daily snapshots are justified, as already suggested in the field survey.

## 3. Summary of precedents

| Question | Common practice | Dissent |
|---|---|---|
| Key an instrument by the venue's id? | No. Keep it, but add your own key | None found |
| Use a numeric venue id as a long-term key? | No (Databento warns of daily remapping) | — |
| Parse symbol strings for type or expiry? | No, use fields | — |
| Precision | Store the increment. Nautilus also stores decimals and validates consistency | vn.py: float tick only |
| Numbers | Scaled integers (Nautilus, Databento) | vn.py: float |
| Variable tick sizes | Only Nautilus supports them, via named schemes | The others assume one tick |
| Fees on the instrument | No (Nautilus), or discouraged (CCXT) | CCXT still includes them |
| One class or flat record | Split: classes (Nautilus) vs flat (Databento, CCXT, vn.py) | — |
| Keep the raw response | Yes | — |
| Time-stamped definitions | Yes where backtesting matters | CCXT, vn.py |

## 4. Open questions

1. Nautilus: does its Rust source store `Price` as a signed 64-bit or 128-bit integer by default, and what range does that cap at? The docs give the scale but not the integer width.
2. Nautilus: how do its Kraken, Binance and Interactive Brokers adapters map venue fields into `CurrencyPair`? That is a direct worked example of the mappings in the field survey, and it should be read before finalising yours.
3. Databento: how are `instrument_id` reassignments recorded in the definition stream itself, as opposed to the symbology endpoint?
4. LEAN: how are delisted symbols handled?
5. CCXT: what numeric type do `precision` and `limits` carry in each language?
6. Which of Hummingbot's `TradingRule`, ib_async's contract handling, and Zipline's asset objects add anything the five above do not?

## Sources

- [CCXT Manual (raw)](https://raw.githubusercontent.com/ccxt/ccxt/master/wiki/Manual.md)
- [CCXT documentation](https://docs.ccxt.com/)
- [NautilusTrader: instruments](https://nautilustrader.io/docs/latest/concepts/instruments/)
- [NautilusTrader: value types](https://nautilustrader.io/docs/latest/concepts/value_types/)
- [NautilusTrader: `currency_pair.rs`](https://raw.githubusercontent.com/nautechsystems/nautilus_trader/develop/crates/model/src/instruments/currency_pair.rs)
- [Databento: instrument definitions](https://databento.com/docs/schemas-and-data-formats/instrument-definitions)
- [Databento: symbology](https://databento.com/docs/standards-and-conventions/symbology)
- [QuantConnect LEAN: security identifiers](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/security-identifiers)
- [vn.py: `object.py`](https://raw.githubusercontent.com/vnpy/vnpy/master/vnpy/trader/object.py)
