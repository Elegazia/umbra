# ADR 0001: Brokers, execution venues and market-data sources

- Status: Accepted
- Date: 2026-11-07 (revised 2026-11-15)
- Deciders: Project owner
- Related: [ADR 0002](0002-instrument-identity-and-definition.md), ADR 0007

## Context

Umbra ingests market data, supports research on it, and will later paper-trade and trade live. Its scope is any listed instrument: equities, ETFs, futures and crypto. Crypto is the first data source because good crypto market data is free and trades around the clock, but crypto does not define the scope.

Two choices follow from this, and they are kept separate. The first is where orders are placed (execution venues). Placing orders involves legal, custody and counterparty risk, so an execution venue must be usable from Singapore under the rules of the Monetary Authority of Singapore (MAS). The second is where prices, trades and order books come from (market-data sources). Reading public market data needs no account and no custody, so a source can be used for research even if the venue cannot be used for trading.

Keeping the two apart is part of this decision. In crypto, price discovery happens mostly on venues that cannot be used for trading from Singapore, while the venues that can be used tend to follow their prices.

The decision is needed now because the first ingestion adapters are being written and the data model (ADR 0002) depends on which venues and asset classes are in scope.

## Decision drivers

Hard constraints. An option that fails one of these is rejected.

- D1. Regulatory eligibility for execution. The venue holds the MAS licence for the service it provides to Singapore users: a Capital Markets Services (CMS) licence for securities and futures brokerage, or a Major Payment Institution (MPI) licence for digital payment token (DPT) services. Venues on the MAS Investor Alert List are excluded.
- D2. Programmatic access. Documented APIs for market data and order management, with streaming (WebSocket or equivalent) for live data.
- D3. A pre-production environment. Paper trading or a sandbox, so the order path can be tested without real money.

Preferences, in order:

- D4. Asset-class coverage. Equities, ETFs, futures and crypto with as few accounts and integrations as possible.
- D5. Research data quality. Full-depth order books with sequence numbers or checksums, exchange timestamps, and long free or cheap history.
- D6. Cost. Low fixed monthly costs, with usage-based pricing preferred to subscriptions.
- D7. API maturity and operational risk. Stable, versioned APIs, official client libraries or widely used open-source adapters, and a good operating record.

## Considered options

Licence facts change over time. The MAS Financial Institutions Directory is the authority. Facts found only in secondary sources are marked "(secondary)".

### Execution: equities, ETFs and futures

E1. Interactive Brokers Singapore (IBKR). A global broker operating through Interactive Brokers Singapore Pte. Ltd. It offers US, Singapore and Hong Kong equities and ETFs, options, CME and SGX futures, and FX, through the TWS API (a socket protocol with official C++, Java and Python clients) and a REST Web API.

- Good, because it is licensed in Singapore and is an SGX derivatives trading and clearing member (secondary) (D1).
- Good, because both APIs are documented and stream market data (D2).
- Good, because one account covers every non-crypto asset class in scope (D4), including CME micro Bitcoin and Ether futures, which give regulated crypto exposure without offshore leverage.
- Good, because there is no platform fee for API use (D6), and the API has a large open-source ecosystem, including `ib_async` and NautilusTrader's adapter (D7).
- Bad, because the paper account is available only alongside a funded live account, and it receives real-time data only if the live user's subscriptions are shared with it (D3).
- Bad, because market data is billed per exchange (D6). Non-consolidated US data is free; SGX equities cost about SGD 10 a month.
- Bad, because the TWS API needs a running gateway process, which is one more component to operate (D7).
- Neutral: its data is good enough for execution but it is not a historical tick archive (D5). Futures and other complex products require MAS's knowledge assessment (see Background).

E2. Saxo Markets Singapore. Saxo Capital Markets Pte. Ltd. is regulated by MAS. Its OpenAPI is a REST API with streaming over SignalR/WebSockets, OAuth 2.0 authentication and a simulation (demo) environment.

- Good, because it is regulated by MAS, according to Saxo's own disclosure (D1), and has documented REST and streaming APIs (D2).
- Good, because its demo environment may not require a funded account (D3).
- Good, because it covers equities, ETFs, futures, options, FX and CFDs (D4).
- Bad, because the fee schedule and API terms for individual Singapore clients were not found (D6), and trading costs are generally higher than at IBKR (secondary).
- Bad, because its open-source ecosystem is much smaller than IBKR's, although the API has existed since 2015 (D7).

E3. moomoo Singapore and Tiger Brokers Singapore. Retail brokers holding MAS CMS licences (secondary). moomoo offers OpenAPI through a local or cloud gateway (OpenD); Tiger offers Tiger Open API.

- Good, because both document streaming APIs, and moomoo charges nothing extra for trading through OpenAPI (D2).
- Good, because commissions are competitive, for example SGX equities at 0.03% plus a platform fee, minimum S$0.99 each (secondary) (D6).
- Bad, because paper trading exists in the apps but API support for it was not confirmed (D3).
- Bad, because futures coverage is narrower than IBKR's (D4), and Tiger bills API quote permissions separately (D5).
- Bad, because the English-language developer ecosystem is small, and moomoo positions OpenAPI for institutional and technically advanced users (D7).

E4. Alpaca. An API-first US broker. In late September 2026 Alpaca Securities Pte. Ltd. received in-principle approval from MAS for a CMS licence. Alpaca states that this does not yet allow it to provide brokerage services, and that it is aimed at institutional partners. It fails D1 and is reconsidered if a full licence is granted and individual accounts are offered.

### Execution: crypto spot

MAS rules prohibit DPT service providers from offering retail customers financing, margin or leverage, or trading incentives. Every option below is therefore spot-only for a retail account. Leveraged crypto exposure comes from regulated futures through E1.

C1. Coinbase (Coinbase Singapore Pte. Ltd.).

- Good, because it holds an MPI licence for DPT services (secondary; sources disagree on the date it was granted) (D1).
- Good, because the Advanced Trade API provides REST and WebSocket access (D2). Spot trading is unaffected by the move of international perpetuals off this API, completed on 1 October 2026.
- Good, because its Level 2 WebSocket channel provides the full book with sequence numbers, so it can also be recorded for research (D5).
- Good, because the API is mature and has official SDKs (D7).
- Bad, because no primary source confirms that API trading is enabled for Singapore accounts (D2), and sandbox availability is not confirmed (D3).
- Bad, because retail fees are higher than at the largest offshore venues (D6).

C2. OKX SG (OKX SG Pte. Ltd.).

- Good, because it holds an MPI licence covering DPT services, reported from 2024 (secondary) (D1).
- Good, because OKX API v5 provides REST and WebSocket access, and OKX SG's help centre documents API key creation (D2).
- Good, because it has a demo-trading environment with separate API keys (D3).
- Good, because the API is mature and widely supported, including by CCXT and NautilusTrader (D7).
- Bad, because the help centre does not state whether every product is enabled for Singapore accounts (D2), and the Singapore product range is narrower than the global platform's.

C3. Crypto.com (Foris DAX Asia Pte. Ltd.). It holds an MPI licence for DPT services, and derivatives and leverage are not offered through the Singapore entity (both secondary). The exchange API (REST, WebSocket, FIX) is mature. API terms specific to Singapore accounts were not found, so it is unclear which entity's terms govern API use.

C4. Independent Reserve (Independent Reserve SG Pte. Ltd.). It holds an MPI licence for DPT services (secondary) and publishes public and private REST APIs and a real-time WebSocket feed. The only official client is for .NET; Python clients are third-party. Liquidity is thinner than at the large venues.

The following venues were rejected for execution because they fail D1:

| Venue | Reason |
|---|---|
| Kraken | Kraken's support page states that it is not licensed or regulated in Singapore, and the MAS directory has no entry for its operating company (secondary report of the search). |
| Binance | On the MAS Investor Alert List since 2021. |
| Bybit | Added to the MAS Investor Alert List on 17 June 2026. |

### Market-data sources

These are used for research only and need no account or custody.

| Source | Covers | Quality (D5) | Cost (D6) |
|---|---|---|---|
| Kraken WebSocket v2 | Live crypto trades and L2 book | CRC32 checksum on the top of the book, so every update can be verified | Free |
| Binance public streams | Live crypto trades and book | Highest crypto spot volume; usually leads price discovery | Free |
| data.binance.vision | Crypto history: trades, aggTrades, klines | Bulk files with `.CHECKSUM` files; years of history | Free |
| Coinbase WebSocket | Live crypto trades and L2 book | Sequence numbers; same venue as C1 | Free |
| IBKR market data | Live equities and futures | Execution-grade, not an archive | Per-exchange subscriptions |
| Databento | US equities, CME futures (history from 2010), full order-book schemas | Exchange-native timestamps; normalised formats | Historical data pay-as-you-go per GB; CME Standard subscription $199 a month for new customers from 22 June 2026 |
| Massive (formerly Polygon) | US equities | Good coverage | About $29 a month for Starter (secondary) |

## Decision

1. Equities, ETFs and futures are executed through IBKR (E1).
2. Crypto spot is executed through Coinbase (C1), provided that Advanced Trade API access is confirmed for Singapore accounts. If it is not, OKX SG (C2) is used, and Independent Reserve (C4) after that.
3. Leveraged crypto exposure is taken only through CME micro Bitcoin and Ether futures on IBKR, never through offshore perpetuals.
4. Research data comes from Kraken WebSocket v2, Binance public streams and data.binance.vision for crypto (free, no account), with the Coinbase WebSocket recorded alongside; Databento pay-as-you-go for non-crypto history; and IBKR for live non-crypto data.

IBKR is the only option that meets D1 to D3 and covers every non-crypto asset class in one account (D4). It also has the most mature API ecosystem (D7) and no API fee (D6). The cost is that a funded account is needed before paper trading.

Coinbase combines a licence (D1), a mature API (D7) and a full-depth feed that can also be recorded for research (D5). Whether its API can be used from a Singapore account (D2) is not yet verified, so the decision is conditional and the fallback is named in advance.

Keeping research data separate from execution lets Umbra learn from the venues where crypto price discovery happens (Binance, Kraken) without holding an account there. It also makes lead–lag research possible: a signal from the leading venue, executed on the licensed one.

Databento's pay-as-you-go pricing suits occasional historical pulls (D6). A subscription is worth it only above roughly 90 GB a month (secondary estimate).

### Rollout

| Phase | Data | Execution |
|---|---|---|
| 1. Data | Kraken WS v2, Binance streams, data.binance.vision, Coinbase WS | None |
| 2. Research | Add Databento pay-as-you-go (CME micro futures, US equities) | Open and fund the IBKR account |
| 3. Paper | Add IBKR live data, shared with the paper account | IBKR paper; Coinbase sandbox or OKX SG demo |
| 4. Live | Unchanged | IBKR for equities and futures; Coinbase (or OKX SG) for crypto spot |

## Consequences

One broker integration covers every non-crypto asset class. The research data pipeline needs no accounts or credentials. Leveraged crypto exposure stays inside a regulated market.

There are two execution integrations (IBKR and Coinbase) with different APIs, order models and failure modes, so a venue-neutral order interface is needed. Paper trading on IBKR needs a funded account and a data subscription shared with the paper user. Futures can be traded only after passing MAS's knowledge assessment.

Coinbase API access for Singapore accounts must be verified before Phase 3. If it is unavailable, OKX SG is used, then Independent Reserve.

The trading signal reads Binance and Kraken but executes on Coinbase, whose lag behaviour may differ from the venues studied. This is measured on recorded data before any strategy depends on it.

Most licence facts come from secondary sources. Each venue is checked on the MAS Financial Institutions Directory before funds are deposited.

Recording public feeds for private research is assumed to be acceptable. Each venue's market-data terms are re-read before any derived data is published.

Follow-up work:

- The record schema and the `DataSource` and `Broker` interfaces, since split into [ADR 0002](0002-instrument-identity-and-definition.md), ADR 0006 and ADR 0007. Because the scope spans asset classes and venues, records need an asset class and a venue in their key; that design belongs to ADR 0002.
- The hosting region (Singapore, or Tokyo to be closer to the Asian venues) is an operations decision for the deployment milestone.

### Revisit when

- Kraken obtains an MAS licence (consider it for crypto execution; its checksummed book is the best-instrumented feed);
- Alpaca receives a full CMS licence and offers individual accounts (compare E4 with E1 on API design and cost);
- the Coinbase API turns out to be unavailable or restricted for Singapore accounts (switch crypto execution to OKX SG in a superseding ADR);
- IBKR market-data or platform costs exceed the budget for two consecutive months (evaluate E3);
- MAS changes the retail DPT rules, for example on leverage, or the complex-products framework (reassess decision 3);
- monthly Databento historical spend exceeds the subscription price (move to a subscription).

## Background

### How Singapore regulates the activities in scope

Securities and futures are regulated under the Securities and Futures Act. A broker dealing in them for customers needs a Capital Markets Services (CMS) licence. IBKR, Saxo, moomoo and Tiger hold one; Alpaca has only in-principle approval.

Digital payment tokens (cryptocurrencies) are regulated under the Payment Services Act 2019. A business that buys, sells or exchanges DPTs for customers needs a Major Payment Institution (MPI) licence with DPT services in scope; smaller businesses may hold a Standard Payment Institution licence. An in-principle approval is not a licence. It means MAS may grant one once its conditions are met.

MAS guidelines protect retail DPT customers: no financing, margin or leverage, no trading incentives, and no public promotion. This is why licensed venues offer only spot trading to retail customers.

Since 30 June 2025, Singapore-incorporated firms that provide digital token services only to overseas customers also need a licence, as Digital Token Service Providers (DTSP) under the Financial Services and Markets Act. The rule applies to providers rather than their customers, but it explains why several exchanges left Singapore.

The MAS Investor Alert List names entities that may have been wrongly perceived as licensed by MAS. It is a warning rather than a ban on use, but it rules a venue out as a counterparty for this project.

Retail investors must pass a knowledge assessment before trading complex products such as futures. This used to be the Customer Account Review (CAR) for listed products and the Customer Knowledge Assessment (CKA) for unlisted ones. In May 2026 MAS announced that "Specified Investment Products" will be renamed "complex products" and that the CAR and CKA will be merged into one assessment.

### Brokers, exchanges and data vendors

An exchange runs the matching engine and the order book (CME, SGX, Binance). A broker gives customers access to one or more exchanges, holds their assets and manages risk (IBKR). In crypto, the exchange usually plays both roles: it matches orders and also holds customer assets, which is why licensing and counterparty risk matter so much. A data vendor redistributes exchange data in a normalised form (Databento). Exchanges charge for their data, and vendors pass the cost on.

### Consolidated and non-consolidated US equity data

US equities trade on more than a dozen exchanges. The consolidated feed (the SIPs) combines every exchange's best quotes into the National Best Bid and Offer (NBBO). Brokers can give retail customers a cheaper non-consolidated feed from one exchange or a subset of them, such as Cboe One or IEX. It reflects only part of the market's liquidity, which is acceptable for research at minute resolution but not for microstructure work.

### Paper trading and sandboxes

A paper account simulates fills against live market data. It does not model queue position or market impact, so paper fills are optimistic. A sandbox is a separate environment, often with synthetic or static data, for checking that requests are well formed. Neither measures real execution quality, which needs small live trades. This is why live trading is a separate phase in the rollout.

### Price discovery and lead–lag

When one asset trades on several venues, new information usually shows up first on the most liquid one, and the others follow with a delay. This is price discovery; in crypto spot, Binance usually leads. A lead–lag strategy reads the leading venue and trades on a lagging one before it catches up. It only works if the delay is longer than the system's own latency plus trading costs, which is why Umbra measures latency at every stage.

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Kraken: where is Kraken licensed or regulated](https://support.kraken.com/articles/where-is-kraken-licensed-or-regulated) | Primary | 2026-11-07 |
| [Asia Forex Mentor: Kraken review 2026](https://www.asiaforexmentor.com/kraken-review/) | Secondary | 2026-11-15 |
| [Asia Forex Mentor: OKX review 2026](https://www.asiaforexmentor.com/okx-review/) | Secondary | 2026-11-15 |
| [OKX SG: API FAQ](https://www.okx.com/en-sg/help/api-faq) | Primary | 2026-11-15 |
| [The Edge: Coinbase awarded MPI licence](https://theedgemalaysia.com/node/684508) | Secondary | 2026-11-15 |
| [Coinbase Developer Platform: changelog](https://docs.cdp.coinbase.com/coinbase-app/introduction/changelog) | Primary | 2026-11-15 |
| [Coinbase Advanced Trade API reference](https://docs.cdp.coinbase.com/api-reference/advanced-trade-api) | Primary | 2026-11-15 |
| [Fintech News SG: licensed crypto providers in Singapore (2026)](https://fintechnews.sg/latest-list-licensed-cryptocurrency-providers-in-singapore/) | Secondary | 2026-11-15 |
| [Capital Markets SG: Independent Reserve SG](https://capitalmarkets.sg/crypto/independent-reserve-sg-pte-ltd) | Secondary | 2026-11-15 |
| [Independent Reserve: API](https://www.independentreserve.com/API) | Primary | 2026-11-15 |
| [Finance Magnates: Bybit added to MAS alert list](https://www.financemagnates.com/cryptocurrency/exchange/singapore-adds-bybit-to-alert-list-alongside-binance-and-kucoin/) | Secondary | 2026-11-15 |
| [Alpaca: MAS in-principle approval](https://alpaca.markets/blog/alpaca-secures-mas-in-principle-approval-to-launch-brokerage-services-in-singapore/) | Primary | 2026-11-15 |
| [Saxo Singapore: OpenAPI](https://www.home.saxo/en-sg/platforms/api) | Primary | 2026-11-15 |
| [moomoo: What is OpenAPI](https://www.moomoo.com/sg/support/topic3_436) | Primary | 2026-11-15 |
| [Tiger Open API: fees and quote permissions](https://quant.itigerup.com/openapi/en/cpp/permission/feePermission.html) | Primary | 2026-11-07 |
| [IBKR Singapore: market data pricing](https://www.interactivebrokers.com.sg/en/pricing/market-data-pricing.php) | Primary | 2026-11-07 |
| [IBKR: market data sharing](https://www.interactivebrokers.com/docs/general/market-data-subscriptions/market-data-users/market-data-sharing) | Primary | 2026-11-07 |
| [IBKR Web API: trading access for individuals](https://www.interactivebrokers.com/docs/web-api/trading/getting-started/trading-access-for-individuals.md) | Primary | 2026-11-07 |
| [Broker Listings: does Interactive Brokers have paper trading?](https://brokerlistings.com/reviews/interactive-brokers/does-interactive-brokers-have-paper-trading) | Secondary | 2026-11-15 |
| [Databento: pricing](https://databento.com/pricing) | Primary | 2026-11-15 |
| [Databento: updates to subscription pricing](https://databento.com/blog/updates-to-subscription-pricing) | Primary | 2026-11-15 |
| [NexusFi: Databento futures market data](https://nexusfi.com/a/data/databento-futures-market-data) | Secondary | 2026-11-15 |
| [API Cost Calc: Polygon/Massive pricing](https://apicostcalc.com/blog/polygon-massive-rebrand-api-pricing.html) | Secondary | 2026-11-07 |
| [4Pillars: Singapore crypto regulation 2026](https://policy.4pillars.io/singapore) | Secondary | 2026-11-15 |
| [MoneySense: understanding specified investment products](https://www.moneysense.gov.sg/investments/understanding-specified-investment-products/) | Primary | 2026-11-15 |
| [Allen & Gledhill: MAS response on PHS and complex products framework](https://www.allenandgledhill.com/sg/publication/articles/33040/mas-responds-to-feedback-on-proposed-enhancements-to-product-highlights-sheet-requirements-and-complex-products-framework) | Secondary | 2026-11-15 |
| [Moneysmart: Interactive Brokers Singapore](https://www.moneysmart.sg/online-brokerage/interactive-brokers) | Secondary | 2026-11-15 |
