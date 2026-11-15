# NautilusTrader adapters: how venue fields become instruments

- **Date:** 2026-11-07
- **Purpose:** Worked examples for [ADR 0002](adr/0002-instrument-identity-and-definition.md). NautilusTrader's Kraken, Binance, Coinbase and Interactive Brokers adapters each convert a venue's instrument data into one common instrument model. This is a direct precedent for the mapping Umbra has to write. Companion to [instrument-definition-field-survey.md](instrument-definition-field-survey.md) (what the venues provide) and [open-source-instrument-model-survey.md](open-source-instrument-model-survey.md) (how projects model instruments).
- **Status:** Survey, not a decision.

## How this was checked

The `develop` branch of [nautechsystems/nautilus_trader](https://github.com/nautechsystems/nautilus_trader) was downloaded file by file and read directly, not summarised. Line numbers will drift as the branch changes.

| Adapter | File read | Lines |
|---|---|---|
| Kraken | `crates/adapters/kraken/src/common/parse.rs` | 3,049 |
| Binance | `crates/adapters/binance/src/common/parse.rs`, `common/instruments.rs` | 2,976 and 289 |
| Interactive Brokers | `crates/adapters/interactive_brokers/src/providers/parse.rs`, `providers/instruments.rs`, `common/symbology.rs` | 1,392, plus two more |
| Coinbase | `crates/adapters/coinbase/src/http/parse.rs` (spot section), `provider.rs` | 2,698 and 599 |

**Limits.** Only the instrument-conversion code was read, not the data or execution paths. Statements that something is "not mapped" come from searching those files, so a field could be handled in code I did not open. Nothing was run. The adapters are Rust and licensed under LGPL v3 (the file headers say so). Understanding the approach is fine; copying code into Umbra would carry licence obligations.

---

## 1. Kraken

**Identity.** Two names, as in every project surveyed:
- `instrument_id` symbol is Kraken's `wsname` (falling back to `altname`) with each side normalised, so `XBT/USDT` becomes `BTC/USDT`.
- `raw_symbol` is the REST pair key (`XBTUSDT`), kept for talking to the venue.

**Asset codes are an explicit table, not a rule.** The adapter maps `XXBT` and `XBT` to `BTC`, `XXDG` and `XDG` to `DOGE`, `XETH` to `ETH`, `ZEUR` to `EUR`, and so on, in a hand-written list. The code comment explains why: Kraken prefixes some legacy assets with `X` or `Z`, "but the prefix is not a rule: `XTZ`, `ZRX`, `XAUT` and others legitimately begin with those letters." Anything not in the table passes through unchanged.

> This corrects the loose wording in the field survey. A normalisation step must be a lookup table, never "strip a leading X". Stripping would turn `XTZ` into `TZ`.

**Price and size grids.**
- `price_increment` comes from `tick_size`. `price_precision` is then taken **from the increment**, not from `pair_decimals`.
- `size_increment` is `10^-lot_decimals`. So the guess I flagged in the field survey is what Nautilus does. It is Nautilus's convention, not something Kraken documents.
- `min_quantity` comes from `ordermin`.
- I found no use of `costmin` or `status` in the spot parser, so minimum notional and trading status are not carried onto the instrument.
- Fees are not set.

**`pair_decimals` is kept only when it matters, and the reason is the order-book checksum.** If `pair_decimals` is finer than the tick precision, it is stored in the instrument's `info` map. If it is coarser, it is dropped. The comment explains: "rendering a price at fewer decimals than its precision truncates rather than rounds, so hashing at that scale would mismatch every message, while the tick-size precision reproduces the venue's digits." This bears directly on order-book reconstruction (CRC32 verification of every book update): **which decimal count you use when formatting prices for the checksum decides whether every message mismatches.**

**Separate parsers per product.** Spot, "tokenized" assets (Kraken's xStocks, equities and ETFs under the spot schema) and futures each have their own function. Kraken Spot's `execution_venue` field can be `international` or `bitnomial_exchange`, which is worth noticing when two order books share a pair name.

**Fixed-point limits are a real constraint.** Nautilus has a standard build with 9 decimal places of precision and a "high-precision" build with 16. Its own test fixture for a Kraken futures contract, `PF_PEPEUSD`, has `tick_size` 0.0000000001 (ten decimals). In the standard build, parsing it **fails**: "tick_size 0.0000000001 requires precision 10, but this build supports at most 9". A tick finer than the build supports is rejected, not rounded.

**Quantity steps can be larger than 1.** The same fixture has `contractValueTradePrecision` of -3, meaning the instrument trades in multiples of 1000. Nautilus produces `size_increment = 1000` with `size_precision = 0`. A schema that assumes quantity steps are 10^-n would be wrong here. (This is the Kraken Futures API, which Singapore retail users cannot use per ADR 0001, but it shows what a venue can send.)

## 2. Binance

**Identity.** Spot is `BTCUSDT.BINANCE` (the symbol unchanged). Futures get a product suffix, e.g. `ETHUSDT-PERP.BINANCE`. `raw_symbol` is the venue symbol.

**Only trading symbols become instruments.** If `status` is not `TRADING`, the parser fails with "Symbol ... is not trading", for both the JSON and the binary (SBE) responses. Status is a **gate**, not a stored field.

**Required filters.** A symbol without `PRICE_FILTER` or `LOT_SIZE` is an error, as is a `tickSize` of zero.

| Instrument field | Source |
|---|---|
| `price_increment`, `price_precision` | `PRICE_FILTER.tickSize` (precision taken from the increment) |
| `size_increment`, `size_precision`, `lot_size` | `LOT_SIZE.stepSize` |
| `min_price`, `max_price` | `PRICE_FILTER.minPrice`, `maxPrice` (optional) |
| `min_quantity`, `max_quantity` | `LOT_SIZE.minQty`, `maxQty` (optional) |
| `min_notional`, `max_notional` | Spot: `NOTIONAL` filter. Futures: `MIN_NOTIONAL` filter. |

The precision fields `baseAssetPrecision` and similar are **not used**. This confirms the trap in the field survey: the real step comes from the filters.

**Invented defaults.** Spot sets margin to 1.0 ("Spot has no leverage"). USD-M futures hard-code a maintenance margin of 0.1. These are the adapter's assumptions, not Binance data.

**Fees are loaded separately** (the functions are named `..._with_fees`), not from the instrument endpoint.

**Two wire formats, one result.** Binance sends instrument data as JSON or as binary SBE. The adapter has parallel parsers that produce the same `CurrencyPair`. For Umbra, that supports parsing at the edge into one internal form.

**Selection.** The provider can load everything, a list of ids, or filter by symbol, base, quote or contract type. Values are upper-cased, duplicates are rejected, and a filter that makes no sense for the product type (e.g. contract type on spot) is an error.

## 3. Interactive Brokers

**The tick-size problem is not solved here.** Every IB parse function sets the price increment from `details.min_tick`, a single number:

```
price_increment = Price::new(details.min_tick, tick_size_to_precision(min_tick))
```

I searched the IB parser and provider files for `market_rule` and found nothing. As the field survey noted, IBKR's `minTick` is only the smallest increment seen on any exchange or price, so this adapter's increment can be wrong for validating individual prices. Nautilus has a `tick_scheme` field that could express variable ticks, but in the files I read the IB adapter does not use it.

**Identity is symbol-based, and configurable.** The `conId` is not part of the id. Two symbology methods exist:
- `IB_SIMPLIFIED`: `AAPL.SMART`, `EUR/USD.IDEALPRO`, `ESM23.GLOBEX`, options as `AAPL230120C00150000.SMART`, indices as `^SPX.SMART`.
- `IB_RAW`: `AAPL=STK.SMART`, `ESM23=FUT.GLOBEX`, with the security type spelled out.

The venue comes from the contract's exchange or primary exchange, optionally converted to a market identifier code. For stocks listed on a primary exchange, the primary exchange wins over `SMART`. For crypto, the venue is `PAXOS` unless the contract says `ZEROHASH`.

**The whole IB contract is stored.** The full `Contract` goes into the instrument's `info` as JSON, along with `priceMagnifier`. That is how `conId` and other IB-specific fields survive.

**Per-type parsers.** One function each for stock, forex, crypto, futures, options, futures options, index, CFD, commodity and bond. Unsupported types are an error.

**Shortcuts visible in the code:**
- Equity `lot_size` is hard-coded to 100.
- Option multiplier defaults to 100, other multipliers to 1, when unparseable (with a warning).
- Futures are classed as the `Index` asset class, and unknown security types default to `Equity`.
- Option strikes are divided by `priceMagnifier`.

**Crypto size rules.** `size_increment` is `sizeIncrement`, else `minSize`, else a fallback of 0.00000001 (that fallback is Nautilus's, not IBKR's). Size precision is the larger of the precision of `minSize` and of the increment. So this adapter did not find fixed values for PAXOS coins either; it depends on what the contract details return.

**Expiry handling.** Futures expiry is turned into a UTC nanosecond timestamp by combining the expiry string, the contract's time zone, and the last session close. An expiry is a local calendar date with a time zone, not just a date.

## 4. Coinbase

**Identity.** `product_id` (e.g. `BTC-USD`) is used for both the instrument symbol and `raw_symbol`. No normalisation is needed because Coinbase already uses standard codes.

**Grids.** The venue sends increments as strings, and the adapter derives precision from them (`precision_from_increment` on `price_increment` and `base_increment`). `base_min_size` and `base_max_size` become min and max quantity. Empty or `"0"` values are treated as absent.

**Fees are loaded separately** (the code comment says maker fee comes "via transaction_summary").

**Workarounds.** For Coinbase futures with an empty `base_currency_id`, the base currency is taken from the first word of `display_name`. Unsupported product types and non-crypto futures are skipped at load time.

**Not set in the spot parser I read:** minimum notional. (I did not search the other Coinbase files.)

---

## 5. Patterns across the four adapters

| Pattern | Kraken | Binance | Coinbase | IBKR |
|---|---|---|---|---|
| Increment is the source of truth, precision derived from it | Yes | Yes | Yes | Yes |
| Quantity step from the venue's own field, not asset precision | `lot_decimals` as 10^-n | `stepSize` | `base_increment` | `sizeIncrement`, then `minSize` |
| Venue symbol kept as `raw_symbol` | Yes | Yes | Yes | Yes (`local_symbol`) |
| Internal id differs from venue symbol | Yes (normalised) | No (spot) | No | Yes (configurable) |
| Non-trading instruments | Not checked in spot parser | Rejected | Unsupported types skipped | Not checked |
| Fees on the instrument | No | No | No | No |
| Raw data kept in `info` | Partly (`pair_decimals`) | No | No | Full contract JSON |
| Invented defaults | None seen | Margins | None seen | Lot size, multipliers, asset class |
| Variable tick size | No | No | No | **No (uses `min_tick`)** |

## 6. Implications for ADR 0002 (options, not decisions)

1. **Normalise asset codes with an explicit table**, per venue, and pass unknown codes through unchanged. Log codes that fall through, so a new asset does not silently keep a venue spelling.
2. **Derive price and quantity precision from the increment.** Do not trust the venues' decimal-count fields. When you must keep a decimal count (Kraken's `pair_decimals` for the checksum), keep it as its own named field.
3. **Decide the book-checksum price scale before building the order book.** Nautilus's finding suggests formatting prices using the tick precision, with `pair_decimals` only when it is finer. Verify this against Kraken's checksum documentation and your own recorded data, not just this comment.
4. **Quantity steps are not always 10^-n.** A schema holding a "number of decimals" for quantity cannot represent a step of 1000. Store the step itself.
5. **Pick a fixed-point scale knowing real instruments exceed 1e-9.** A Kraken futures contract with a 1e-10 tick is rejected by Nautilus's standard build. If you adopt scaled integers, choose the scale from the finest tick or step you will store, and make out-of-range definitions fail loudly instead of rounding.
6. **Decide whether not-trading instruments are rejected or stored.** Nautilus rejects them because it is a live engine. A recorder that needs history for delisted or halted pairs should store them with a status. This is the one place where the precedent argues *against* following Nautilus.
7. **IBKR variable ticks are an open problem even in Nautilus.** If you need correct tick validation for IBKR equities, you will need market rules, and no adapter surveyed does that.
8. **Keep the raw IB contract.** Nautilus keeps the whole `Contract` as JSON. That preserves `conId`, which the symbol-based id drops.
9. **Do not invent defaults silently.** The invented margins, lot sizes and multipliers above are where the adapters guess. In a research database, prefer null plus a flag.
10. **Expiries need a time zone.** Store futures expiry as a UTC timestamp derived from the venue's local date, time zone and session close.

## 7. Answers to earlier open questions

| Earlier question | Answer |
|---|---|
| Field survey Q1: Kraken quantity step | Nautilus uses 10^-`lot_decimals`. Still Nautilus's convention, not documented by Kraken. |
| Field survey Q3: IBKR PAXOS crypto size limits | Not found. Nautilus falls back to 1e-8 when `sizeIncrement` and `minSize` are absent. Needs a live response. |
| Open-source survey Q2: how adapters map venue fields | This document. |
| Open-source survey Q1: integer width of `Price` | Not answered. Only the decimal precision limits were found: 9 in the standard build, 16 in high-precision. |

Still unknown: Nautilus's handling of Kraken's `status` and `costmin` outside the spot parser, and whether any IB code path uses market rules outside the files I read.

## Sources

All from the `develop` branch of [nautechsystems/nautilus_trader](https://github.com/nautechsystems/nautilus_trader):

- [Kraken `common/parse.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/kraken/src/common/parse.rs)
- [Binance `common/parse.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/binance/src/common/parse.rs)
- [Binance `common/instruments.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/binance/src/common/instruments.rs)
- [IB `providers/parse.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/interactive_brokers/src/providers/parse.rs)
- [IB `providers/instruments.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/interactive_brokers/src/providers/instruments.rs)
- [IB `common/symbology.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/interactive_brokers/src/common/symbology.rs)
- [Coinbase `http/parse.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/coinbase/src/http/parse.rs)
- [Coinbase `provider.rs`](https://github.com/nautechsystems/nautilus_trader/blob/develop/crates/adapters/coinbase/src/provider.rs)
