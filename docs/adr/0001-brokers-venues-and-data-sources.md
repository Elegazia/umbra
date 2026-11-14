# ADR 0001: Brokers, venues and data sources

- **Status:** Proposed
- **Date:** 2026-11-07
- **Related:** ADR 0002 (record schema and `DataSource` interface)

## Context

Umbra has to ingest, research and eventually trade any symbol: equities, ETFs, futures and crypto. Crypto comes first because its market data is free and trades around the clock (Kraken and Binance feeds), but it is the first source, not the scope.

Every venue has to be usable from Singapore under MAS rules. That excludes several exchanges that would otherwise be the obvious choice. Brokers that only serve US residents are out too.

Requirements:

- A programmatic API for both data and orders
- Paper trading or a sandbox before live money
- Coverage for stocks, futures and crypto without running five separate brokers
- Low fixed costs: pay-as-you-go where possible
- Data good enough to measure latency honestly and to cross-check venues against each other

## Options considered

### Brokers and crypto venues

| Venue | Asset classes | Singapore status | API | Cost notes | Verdict |
|---|---|---|---|---|---|
| Interactive Brokers (IBKR Pro) | US/SGX/HK stocks, ETFs, options, CME futures, FX | Available | TWS API, Web API | Free non-consolidated US data (Cboe One/IEX); SGX stocks ~SGD 10/mo | Primary broker |
| Coinbase (Advanced Trade) | Crypto spot | MAS MPI licence (PS20200603) | REST, WebSocket, sandbox | International perps left this API in Oct 2026; spot unaffected | Crypto execution |
| OKX SG | Crypto spot; perps after a suitability quiz | Reported MPI licence; one 2026 source says in-principle approval only | Singapore API terms not confirmed | — | Crypto backup |
| Crypto.com | Crypto | MAS MPI licence | FIX, REST, WebSocket | Singapore API terms not confirmed | Crypto backup |
| Kraken | Crypto | Not licensed in Singapore (Kraken's own support page); not geo-blocked | WebSocket v2 with book checksum | — | Data only |
| Binance | Crypto | Restricts Singapore users; on MAS alert list since 2021 | Public streams, data.binance.vision | Free | Data only |
| Bybit | Crypto | MAS Investor Alert List, June 2026; blocks Singapore users | — | — | Avoid |
| Alpaca | US stocks | MAS in-principle approval only (Sept 2026); no direct Singapore accounts yet | Very good | — | Revisit in 2027 |
| moomoo SG / Tiger | US/HK/SG stocks, futures | MAS CMS licence holders | OpenAPI / Open API, no extra trading fee (Tiger bills API market data separately) | — | Fallback if IBKR costs become a problem |

### Data providers

| Source | Covers | Cost | Verdict |
|---|---|---|---|
| Kraken WS v2, Binance public stream, data.binance.vision | Live crypto books and trades, crypto history | Free | Use now |
| IBKR market data | Live stocks, futures, SGX | Bundled with the account | Live non-crypto data |
| Databento | US equities, CME futures back to 2010, order-book data | $125 free credit; historical billed per GB; CME subscription now $199/mo | Non-crypto history, pay-as-you-go only |
| Massive (formerly Polygon) | US stocks | ~$29/mo Starter | Only if Databento per-GB costs get high |

## Decision

**IBKR is the broker for everything except crypto. Coinbase is the crypto execution venue. Kraken and Binance are research data only. Databento supplies non-crypto history.**

| Phase | Data | Execution |
|---|---|---|
| Phase 1 (data) | Kraken WS v2, Binance stream, Binance bulk history | None |
| Phase 2 (research) | Add Databento pay-as-you-go for CME micro futures and US equities | Open and fund an IBKR Pro account |
| Phase 3 (paper) | Add IBKR live data, shared to the paper account | IBKR paper, Coinbase sandbox |
| Phase 4 (live) | Same | IBKR for equities/futures, Coinbase for crypto spot; OKX SG as backup |

Leverage on BTC/ETH goes through CME micro futures on IBKR, not through offshore perps.

## Consequences

- **Data model.** Every record is keyed by `(asset_class, venue, symbol)`. Parquet partitions become `asset_class/venue/symbol/date` instead of `venue/symbol/date`. Trades and order books get separate schemas, since many equity data plans don't include full depth.
- **Interfaces.** One `DataSource` interface and one `Broker` interface, with an adapter per venue. The strategy code doesn't know which venue it is talking to. This also gives agents a single way to look up any instrument.
- **IBKR paper needs a funded live account.** The paper account only exists on a funded IBKR Pro account, and it only gets real-time API data if the live user's subscriptions are shared with it. Paper trading on IBKR therefore needs a funded account first.
- **Lead–lag execution moves from Kraken to Coinbase.** The signal reads Binance (and Kraken) and trades on Coinbase. Coinbase's lag profile may differ from Kraken's, so measure it on recorded data before building on it.
- **VPS location.** Host in Singapore or Tokyo rather than the US. It's closer to the Asian venues and avoids Binance blocking US IPs.
- **Licence status changes.** Most licence facts above come from secondary sources. Confirm each venue on the MAS Financial Institutions Directory before depositing money, and revisit this ADR when Alpaca or Kraken get a Singapore licence.

## Sources

- [Kraken: where is Kraken licensed](https://support.kraken.com/articles/where-is-kraken-licensed-or-regulated)
- [Asia Forex Mentor: Kraken review 2026](https://www.asiaforexmentor.com/kraken-review/)
- [Exchange Catalogue: MAS-licensed exchanges 2026](https://exchangecatalogue.com/best/crypto-exchanges-in-singapore/)
- [CryptoLawMap: Singapore 2026](https://cryptolawmap.com/regulations/singapore/)
- [Coinbase Singapore](https://www.coinbase.com/en-sg)
- [Coinbase Advanced Trade FAQ](https://docs.cdp.coinbase.com/advanced-trade/docs/faq)
- [The Block: OKX fully licensed in Singapore](https://www.theblock.co/post/314088/okx-full-payments-license-in-singapore)
- [CoinGecko: best crypto exchanges in Singapore](https://www.coingecko.com/learn/best-crypto-exchanges-in-singapore)
- [Fintech News SG: licensed crypto providers](https://fintechnews.sg/latest-list-licensed-cryptocurrency-providers-in-singapore/)
- [Finance Magnates: Bybit added to MAS alert list](https://www.financemagnates.com/cryptocurrency/exchange/singapore-adds-bybit-to-alert-list-alongside-binance-and-kucoin/)
- [AMBCrypto: Bybit blocks Singapore users](https://ambcrypto.com/bybit-says-it-blocks-singapore-users-after-mas-investor-alert-listing/)
- [Forkast: Binance restricts Singapore](https://forkast.news/binance-restrict-singapore-from-binance-com/)
- [Alpaca: MAS in-principle approval](https://alpaca.markets/blog/alpaca-secures-mas-in-principle-approval-to-launch-brokerage-services-in-singapore/)
- [moomoo OpenAPI overview](https://open.moomoo.com/api/overview/)
- [Tiger Open API: fees and quote permissions](https://quant.itigerup.com/openapi/en/cpp/permission/feePermission.html)
- [IBKR Singapore market data pricing](https://www.interactivebrokers.com.sg/en/pricing/market-data-pricing.php)
- [IBKR: market data sharing](https://www.interactivebrokers.com/docs/general/market-data-subscriptions/market-data-users/market-data-sharing)
- [IBKR Web API: trading access for individuals](https://www.interactivebrokers.com/docs/web-api/trading/getting-started/trading-access-for-individuals.md)
- [Databento pricing](https://databento.com/pricing)
- [Databento: updates to subscription pricing](https://databento.com/blog/updates-to-subscription-pricing)
- [API Cost Calc: Polygon/Massive pricing](https://apicostcalc.com/blog/polygon-massive-rebrand-api-pricing.html)
- [Investing in the Web: best API brokers](https://investingintheweb.com/brokers/best-api-brokers)
