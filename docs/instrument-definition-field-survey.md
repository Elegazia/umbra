# Instrument definition: field survey (Kraken, Binance, IBKR)

- **Date:** 2026-11-07
- **Purpose:** Input to [ADR 0002](0002-Record-Schema-and-DataSource-Interface). What each venue actually provides for an instrument definition, and where the venues disagree.
- **Status:** Survey, not a decision. Nothing here fixes the schema.

## How this was checked

| Venue | What was read | Caveat |
|---|---|---|
| Kraken | Live `AssetPairs?pair=XBTUSDT` response, plus the REST reference page | Response returned as JSON. Field list matches the reference page. |
| Binance | Live `exchangeInfo?symbol=BTCUSDT` response, plus the filters reference page | The fetch tool summarised the response instead of returning it verbatim, and abbreviated the long `permissionSets` list. Re-run `curl` yourself and keep the raw file. |
| IBKR | TWS API `Contract` and `ContractDetails` reference, `reqContractDetails`, market-rule pages, crypto contract page, Web API contract endpoints | Documentation only. No live response was fetched, because it needs a logged-in gateway. The TWS reference pages say they are deprecated in favour of IBKR Campus, so re-check against a live response once the account exists. |

Not covered: Coinbase (named as the crypto execution venue in ADR 0001).

---

## 1. Kraken: `GET /0/public/AssetPairs`

Observed for `XBTUSDT`:

```
altname "XBTUSDT"   wsname "XBT/USDT"   base "XXBT"   quote "USDT"
aclass_base "currency"   aclass_quote "currency"   lot "unit"
pair_decimals 1   cost_decimals 5   lot_decimals 8   lot_multiplier 1
ordermin "0.00006"   costmin "0.5"   tick_size "0.1"   status "online"
leverage_buy/sell [2..10]   margin_call 80   margin_stop 40
long/short_position_limit 80   fee_volume_currency "ZUSD"
fees []   fees_maker []   execution_venue "international"
```

| Field | Type | Meaning | Use for Umbra |
|---|---|---|---|
| `altname` | string | Alternate pair name | Venue symbol (REST) |
| `wsname` | string | WebSocket pair name | Venue symbol (WebSocket). Needed to subscribe. |
| `base`, `quote` | string | Kraken asset IDs | Normalise, see section 4 |
| `aclass_base`, `aclass_quote` | string | Asset class of each side | Store raw only |
| `tick_size` | string | Minimum price increment | Tick size |
| `pair_decimals` | int | Decimal places for prices | Price scale. Needed for the book checksum. |
| `lot_decimals` | int | Decimal places for volume | Quantity scale |
| `cost_decimals` | int | Decimal places for cost in quote terms | Store raw only |
| `lot_multiplier` | int | Multiplier on lot volume | Store raw only |
| `ordermin` | string | Minimum order size, base currency | Min quantity |
| `costmin` | string | Minimum order cost, quote currency | Min notional |
| `status` | string | `online`, `cancel_only`, `post_only`, `limit_only`, `reduce_only` | Status |
| `execution_venue` | enum | `international` or `bitnomial_exchange` | Store. Two order books can exist under one Kraken name. |
| `lot` | string | **Deprecated** | Ignore |
| `fees`, `fees_maker` | array | **Deprecated as of 8 Sep 2026**, always `[]` | Ignore. Get fees elsewhere. |
| `leverage_*`, `margin_*`, `*_position_limit`, `fee_volume_currency` | various | Margin and fee data | Out of scope for spot data. Store raw only. |

Numbers that are strings (`tick_size`, `ordermin`, `costmin`) must be parsed straight to an exact type, never through a float.

## 2. Binance: `GET /api/v3/exchangeInfo`

Observed for `BTCUSDT`:

```
status TRADING   baseAsset BTC   quoteAsset USDT
baseAssetPrecision 8   quotePrecision 8   quoteAssetPrecision 8
baseCommissionPrecision 8   quoteCommissionPrecision 8
PRICE_FILTER      minPrice 0.01  maxPrice 1000000  tickSize 0.01
LOT_SIZE          minQty 0.00001 maxQty 9000       stepSize 0.00001
MARKET_LOT_SIZE   minQty 0       maxQty 137.63...  stepSize 0
NOTIONAL          minNotional 5  applyMinToMarket true
                  maxNotional 9000000  applyMaxToMarket false  avgPriceMins 5
PERCENT_PRICE_BY_SIDE, ICEBERG_PARTS, TRAILING_DELTA, MAX_NUM_* ...
```

| Field | Meaning | Use for Umbra |
|---|---|---|
| `symbol` | Venue symbol, e.g. `BTCUSDT` | Venue symbol |
| `baseAsset`, `quoteAsset` | Asset names | Base and quote |
| `status` | Trading status. Only `TRADING` was observed. | Status. Collect the other values from the docs. |
| `PRICE_FILTER.tickSize` | Valid prices are multiples of this | Tick size |
| `PRICE_FILTER.minPrice`, `maxPrice` | Allowed price range | Optional |
| `LOT_SIZE.stepSize` | Valid quantities are multiples of this | Quantity step |
| `LOT_SIZE.minQty`, `maxQty` | Quantity range | Min quantity |
| `NOTIONAL.minNotional` | Minimum price × quantity | Min notional |
| `baseAssetPrecision` etc. | Decimal places for asset amounts | **Not** the lot step. See section 4. |
| `orderTypes`, allowed flags, `permissionSets` | What the symbol supports | Store raw only |
| `rateLimits`, `serverTime`, `timezone` | Top-level, not per symbol | Not instrument data. `serverTime` is useful for clock-offset logging. |

Structure to note: `filters` is a **list of objects with different shapes, tagged by `filterType`**, not a fixed set of columns. Parse by type, and expect symbols to carry filters others lack. The docs list both `MIN_NOTIONAL` (older) and `NOTIONAL` (newer). The BTCUSDT response used `NOTIONAL`.

Binance restricts Singapore users (ADR 0001). This is a public, read-only market-data endpoint, but read Binance's terms on data use and record the result in the ADR.

## 3. IBKR: TWS API `Contract` and `ContractDetails`

IBKR has no flat "instrument" record. You describe a contract with a `Contract` object, ask `reqContractDetails`, and get back zero or more `ContractDetails`, each containing a full `Contract`.

**`Contract` fields**

| Field | Meaning |
|---|---|
| `conId` | IBKR's unique contract identifier. The only field the docs call unique. |
| `symbol` | Underlying's symbol |
| `secType` | `STK` (stocks and ETFs), `OPT`, `FUT`, `IND`, `FOP`, `CASH` (forex), `BAG`, `WAR`, `BOND`, `CMDTY`, `NEWS`, `FUND`. The crypto page uses `CRYPTO`, which the main list omits. |
| `exchange` | Destination exchange. May be a router such as `SMART`. |
| `primaryExchange` | Native exchange, resolves ambiguity for smart-routed contracts |
| `currency` | Currency of the contract |
| `localSymbol` | Symbol on the primary exchange. For options, the OCC symbol. |
| `tradingClass` | e.g. `FGBL` |
| `lastTradeDateOrContractMonth` | `YYYYMM` is a contract month. `YYYYMMDD` is a last trading day. |
| `multiplier` | For options and futures |
| `strike`, `right` | Options only |
| `includeExpired` | Needed to query expired futures. Not available for expired options. |
| `secIdType`, `secId` | ISIN, CUSIP, etc. |

**`ContractDetails` fields**

| Field | Meaning | Note |
|---|---|---|
| `minTick` | Smallest increment seen on **any** exchange or price | **Not enough for validation.** See below. |
| `marketRuleIds` | List of rule IDs, in the same order as `validExchanges` | The real tick structure |
| `validExchanges` | Exchanges where orders can be placed | |
| `priceMagnifier` | Scale factor to keep prices consistent across data and orders | |
| `minSize`, `sizeIncrement`, `suggestedSizeIncrement` | Order size limits | Per-coin values for crypto not found in the docs. Read from a live response. |
| `timeZoneId`, `tradingHours`, `liquidHours` | Schedule, as strings | Must be parsed. Crypto trades 24/7. |
| `contractMonth`, `realExpirationDate`, `lastTradeTime` | Futures expiry | |
| `underConId` | Underlying's `conId` | Links a future or option to its underlying |
| `longName`, `marketName`, `stockType` | Descriptive | |
| `evMultiplier`, `aggGroup`, `secIdList` | Misc. | `aggGroup` is the smart-routing group, `-1` if not routable |

**Tick size is a function, not a number.** `minTick` is explicitly the smallest increment encountered anywhere. For tick-aware validation, take `marketRuleIds`, call `reqMarketRule` for each, and receive a list of `PriceIncrement` entries, each a low price edge plus the increment that applies from that price upward. The rule differs per exchange. The docs add that some non-US securities have minimum lot sizes that the API does not expose.

**Crypto:** `secType=CRYPTO`, `exchange=PAXOS` or `ZEROHASH`, `currency=USD` (only USD at the time of the docs), and not every account may use both exchanges. The same coin has a different `conId` on each exchange.

**Web API instead of TWS:** the same data is available over REST, but the key names are not consistent. The `/iserver/contract/{conid}/info` docs show both `conid` and `con_id`, and `maturity_date` in snake_case. `/iserver/secdef/info` uses camelCase such as `maturityDate`. Do not assume shared names across endpoints.

## 4. Where the venues disagree

| Concept | Kraken | Binance | IBKR |
|---|---|---|---|
| Venue symbol | `altname` (REST), `wsname` (WS) | `symbol` | `conId`, with `symbol`, `secType`, `exchange`, `currency`, `localSymbol` |
| Bitcoin as base | `XXBT` (`base`), `XBT` (`wsname`) | `BTC` | `BTC` (crypto) |
| Quote | `quote` | `quoteAsset` | `currency` |
| Tick size | one scalar, `tick_size` | one scalar, `PRICE_FILTER.tickSize` | **A table by price band and exchange** |
| Quantity step | not given. Only `lot_decimals`, so the step is presumably 10^-`lot_decimals`. **Inference, verify.** | `LOT_SIZE.stepSize` | `sizeIncrement` |
| Min notional | `costmin` | `NOTIONAL.minNotional` | not provided |
| Status | five-value enum | status string, only `TRADING` observed | no status field in the pages read |
| Identity key | pair name | symbol | `conId` (many contracts share one `symbol`) |
| Expiry, multiplier | not applicable (spot) | not applicable (spot) | yes |
| Trading hours | always on | always on | timezone and schedule strings |
| Deprecated fields | `lot`, `fees`, `fees_maker` | older `MIN_NOTIONAL` | TWS reference pages themselves |

## 5. Traps found

1. **Three spellings of Bitcoin:** `XXBT`, `XBT`, `BTC`. A normalisation map is mandatory, and it must be kept separate from the raw venue symbol.
2. **Binance precision fields mislead.** `baseAssetPrecision` is 8, but the real quantity step is 0.00001. Use the filters, not the precision fields.
3. **Kraken has no explicit lot step.** Treat the 10^-`lot_decimals` reading as a guess until checked against a live order or Kraken's documentation.
4. **Binance filters are heterogeneous.** One flat column per filter will not work across symbols.
5. **IBKR identity is not the symbol.** A symbol like `ES` returns many contracts, one per expiry. The key must be `conId`, and the schema needs the expiry fields.
6. **IBKR tick size depends on price.** A scalar `tick_size` column cannot represent it, and a validator that uses one will reject valid prices.
7. **Definitions change.** Kraken deprecated fee fields on 8 September 2026, so even the field set is not stable. This supports versioning definitions (valid_from, valid_to) or daily snapshots.
8. **Keep the raw response.** Fields you do not model today (margin levels, order types, filters) cost nothing to keep and let you reprocess later.

## 6. Implications for the schema (options, not decisions)

- **Common core** that all three venues can fill: internal id, asset class, venue, venue symbol, base, quote, instrument type, status, min quantity, quantity step, price increments, fetch time, validity range.
- **Price increments as a list** of `(low_edge, increment)`. Crypto gets one entry with low edge 0. IBKR gets one entry per price band. This fits all three venues, whereas a scalar `tick_size` fits only two.
- **Derivatives fields** (expiry, multiplier, underlying id, strike, right) null for spot.
- **Hours** as a nullable field for IBKR only.
- **A raw column** holding the venue's original response, per row, so no field is lost.
- **Identity:** the natural key `(venue, venue_symbol)` works for Kraken and Binance. For IBKR it needs `conId`, and for crypto on IBKR `conId` plus exchange. A surrogate `instrument_id` hides that difference, which argues for using one.

## 7. Open questions

1. Kraken quantity step: derive from `lot_decimals`, or confirm from documentation?
2. Binance: list the full set of `status` values from the docs, since only `TRADING` was seen.
3. IBKR: what `sizeIncrement` and `minSize` are for PAXOS crypto. Needs a live `reqContractDetails`.
4. IBKR: whether Singapore accounts can use PAXOS or ZEROHASH.
5. IBKR: whether you need the TWS API (C++ client available) or the Web API, since field names differ.
6. Coinbase: survey the product endpoint before the execution phase.
7. Binance terms of use for public data from Singapore.

## Sources

- [Kraken: Get tradable asset pairs](https://docs.kraken.com/api/docs/rest-api/get-tradable-asset-pairs)
- [Kraken: live AssetPairs for XBTUSDT](https://api.kraken.com/0/public/AssetPairs?pair=XBTUSDT)
- [Binance Spot API: filters](https://developers.binance.com/docs/binance-spot-api-docs/filters)
- [Binance: live exchangeInfo for BTCUSDT](https://api.binance.com/api/v3/exchangeInfo?symbol=BTCUSDT)
- [IBKR TWS API: ContractDetails](https://interactivebrokers.github.io/tws-api/classIBApi_1_1ContractDetails.html)
- [IBKR TWS API: Contract](https://interactivebrokers.github.io/tws-api/classIBApi_1_1Contract.html)
- [IBKR TWS API: contract details](https://interactivebrokers.github.io/tws-api/contract_details.html)
- [IBKR TWS API: minimum price increment](https://www.interactivebrokers.com/docs/tws-api/doc/orders/minimum-price-increment)
- [IBKR TWS API: request market rule](https://www.interactivebrokers.com/docs/tws-api/doc/orders/minimum-price-increment/request-market-rule.md)
- [IBKR TWS API: cryptocurrency contracts](https://www.interactivebrokers.com/docs/tws-api/contracts/cryptocurrency.md)
- [IBKR Web API: get info and rules](https://www.interactivebrokers.com/docs/web-api/api-reference/trading/trading-contracts/get-info-and-rules.md)
- [IBKR Web API: contract information by contract id](https://www.interactivebrokers.com/docs/web-api/v1/endpoints/contract/contract-information-by-contract-id.md)
