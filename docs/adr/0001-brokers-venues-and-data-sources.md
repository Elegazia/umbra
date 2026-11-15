# ADR 0001: Brokers, execution venues and market-data sources

- **Status:** Accepted
- **Date:** 2026-11-07 (revised 2026-11-15)
- **Deciders:** Project owner
- **Related:** ADR 0002 (record schema and `DataSource` interface, planned)

## Context

Umbra ingests market data, supports research on it, and will later paper-trade and trade live. Its scope is any listed instrument (equities, ETFs, futures and crypto). Crypto is the first data source because high-quality crypto market data is free and trades continuously, but it does not define the scope.

Two separate choices follow from that:

1. **Execution venues:** where orders are placed. This carries legal, custody and counterparty risk, so the venue must be usable from Singapore under Monetary Authority of Singapore (MAS) rules.
2. **Market-data sources:** where prices, trades and order books come from. Reading public market data does not involve an account or custody, so a source can be used for research even where the venue cannot be used for trading.

Separating the two is itself part of this decision. Price discovery in crypto concentrates on venues that cannot be used for trading from Singapore, while the venues that can be used are often price followers.

The decision is needed now because the first ingestion adapters are being built, and the data model (ADR 0002) depends on which venues and asset classes are in scope.

## Decision drivers

Hard constraints (an option that fails one is rejected):

1. **D1 (hard): Regulatory eligibility for execution.** The execution venue is licensed by MAS for the service it provides to Singapore users: a Capital Markets Services (CMS) licence for securities and futures brokerage, or a Major Payment Institution (MPI) licence for digital payment token (DPT) services. Venues on the MAS Investor Alert List are excluded.
2. **D2 (hard): Programmatic access.** Documented APIs for both market data and order management, with streaming (WebSocket or equivalent) for live data.
3. **D3 (hard): A pre-production environment.** Paper trading or a sandbox, so that the order path can be tested without real money.

Preferences, ranked:

4. **D4: Asset-class coverage per account.** Equities, ETFs, futures and crypto with as few accounts and integrations as possible.
5. **D5: Research data quality.** Full-depth order books with sequence numbers or checksums, exchange timestamps, and long free or cheap history.
6. **D6: Cost.** Low fixed monthly costs; usage-based pricing preferred to subscriptions.
7. **D7: API maturity and operational risk.** Stable, versioned APIs, official client libraries or widely used open-source adapters, and a strong operating record.

## Options considered

Licence facts below are time-sensitive. The MAS Financial Institutions Directory is the authority. Where only secondary sources were found, the fact is marked *(secondary)*.

### Execution: equities, ETFs and futures

#### Option E1: Interactive Brokers Singapore (IBKR)

A global broker operating in Singapore through Interactive Brokers Singapore Pte. Ltd. It provides access to US, Singapore and Hong Kong equities and ETFs, options, CME and SGX futures, and FX, through the TWS API (socket protocol, official C++/Java/Python clients) and a REST Web API.

| Driver | Assessment |
|---|---|
| D1 | Licensed in Singapore and an SGX derivatives trading and clearing member *(secondary)*. Futures and other complex products require MAS's knowledge assessment (see Background). |
| D2 | TWS API and Web API, both documented, with streaming market data. |
| D3 | Paper account with simulated funds, but only on a **funded** live account; paper accounts receive real-time data only if the live user's subscriptions are shared with them. |
| D4 | Every non-crypto asset class in scope, in one account. |
| D5 | Live data adequate for execution; not a historical tick archive. |
| D6 | No platform fee for the API; market data billed per exchange (free non-consolidated US data; SGX equities about SGD 10/month). |
| D7 | Long-established API, with a large open-source ecosystem (for example `ib_async` and NautilusTrader's adapter). The TWS API needs a running gateway process. |

- **Benefits:** one account covers every non-crypto asset class; regulated futures give exposure to crypto (CME micro Bitcoin and Ether futures) without offshore leverage.
- **Costs:** a funded account is needed before paper trading; the gateway adds an operational component; market data is billed per exchange.

#### Option E2: Saxo Markets Singapore

Saxo Capital Markets Pte. Ltd. is regulated by MAS. OpenAPI is a REST API with streaming (SignalR/WebSockets) and OAuth 2.0 authentication, plus a simulation (demo) environment.

| Driver | Assessment |
|---|---|
| D1 | Regulated by MAS (Saxo's own disclosure). |
| D2 | REST and streaming, documented on a developer portal. |
| D3 | Simulation environment available. |
| D4 | Equities, ETFs, futures, options, FX and CFDs. |
| D5 | Execution-grade data, not a research archive. |
| D6 | Not confirmed: the fee schedule and API terms for individual Singapore clients were not found. |
| D7 | Mature (launched 2015), but a much smaller open-source ecosystem than IBKR's. |

- **Benefits:** a clean REST design and a demo environment that may not need funding.
- **Costs:** unknown API terms for individuals; fewer community adapters; generally higher trading costs than IBKR *(secondary)*.

#### Option E3: moomoo Singapore / Tiger Brokers Singapore

Retail brokers holding MAS CMS licences, each with an API: moomoo OpenAPI (through a local or cloud gateway, OpenD) and Tiger Open API.

| Driver | Assessment |
|---|---|
| D1 | MAS-licensed *(secondary)*. |
| D2 | Documented APIs with streaming quotes; moomoo charges nothing extra for trading through OpenAPI. |
| D3 | Paper trading available in the apps; API paper support not confirmed. |
| D4 | US, Hong Kong and Singapore equities, and some futures; narrower futures coverage than IBKR. |
| D5 | Execution-grade data; Tiger bills API quote permissions separately. |
| D6 | Competitive commissions (for example SGX equities at 0.03% plus a platform fee, minimum S$0.99 each, *secondary*). |
| D7 | Smaller English-language developer ecosystem; moomoo positions OpenAPI for institutional and technically advanced users. |

- **Benefits:** a credible fallback if IBKR's costs or terms become a problem.
- **Costs:** narrower derivatives coverage and a weaker API ecosystem.

#### Option E4: Alpaca

An API-first US broker. In late September 2026 Alpaca Securities Pte. Ltd. received **in-principle approval** from MAS for a CMS licence. Alpaca states that this does not yet allow it to provide brokerage services, and its stated target is institutional partners.

- **Fails D1 today.** Revisit when a full licence is granted and individual accounts are offered.

### Execution: crypto spot

MAS rules prohibit DPT service providers from offering retail customers financing, margin or leverage, and from offering trading incentives. Every option below is therefore spot-only for a retail account. Leveraged crypto exposure comes from regulated futures through E1.

#### Option C1: Coinbase (Coinbase Singapore Pte. Ltd.)

| Driver | Assessment |
|---|---|
| D1 | MPI licence for DPT services *(secondary; the reported grant date is inconsistent across sources)*. |
| D2 | Advanced Trade API: REST and WebSocket. Spot is unaffected by the move of international perpetuals off this API (completed 1 October 2026). **Not confirmed from a primary source that API trading is enabled for Singapore accounts.** |
| D3 | Sandbox availability to be confirmed. |
| D5 | Full-depth Level 2 WebSocket channel with sequence numbers. |
| D7 | Mature, well-documented API; official SDKs. |

- **Benefits:** a major regulated venue with a strong API; its order books can also be recorded as a research source.
- **Costs:** Singapore API eligibility unverified; retail fees higher than at the largest offshore venues.

#### Option C2: OKX SG (OKX SG Pte. Ltd.)

| Driver | Assessment |
|---|---|
| D1 | MPI licence covering DPT services *(secondary; reported from 2024)*. |
| D2 | OKX API v5 (REST and WebSocket). OKX SG's help centre documents API-key creation; whether every product is enabled for Singapore accounts is not stated. |
| D3 | Demo-trading environment with separate API keys. |
| D7 | Mature, widely supported API (CCXT, NautilusTrader). |

- **Benefits:** a strong API and demo environment.
- **Costs:** Singapore product scope narrower than the global platform; API terms for Singapore not stated explicitly.

#### Option C3: Crypto.com (Foris DAX Asia Pte. Ltd.)

- MPI licence for DPT services *(secondary)*. Derivatives and leverage are not offered through the Singapore entity *(secondary)*. API terms specific to Singapore accounts were not found. The exchange API (REST, WebSocket, FIX) is mature.
- **Costs:** unclear which entity's terms govern API use for Singapore accounts.

#### Option C4: Independent Reserve (Independent Reserve SG Pte. Ltd.)

- MPI licence for DPT services *(secondary)*. Public and private REST APIs and a real-time WebSocket feed (official). Only an official .NET client; Python clients are third-party.
- **Benefits:** a Singapore-licensed venue with an explicit API.
- **Costs:** thinner liquidity than the large venues; small client ecosystem.

#### Rejected for execution

| Venue | Reason (fails D1) |
|---|---|
| Kraken | Kraken's support page states it is not licensed or regulated in Singapore, and the MAS directory returns no entry for its operating company *(secondary report of the search)*. |
| Binance | On the MAS Investor Alert List since 2021. |
| Bybit | Added to the MAS Investor Alert List on 17 June 2026. |

### Market-data sources (research only; no account, no custody)

| Source | Covers | Quality (D5) | Cost (D6) |
|---|---|---|---|
| Kraken WebSocket v2 | Live crypto trades and L2 book | CRC32 checksum on the top of the book lets every update be verified | Free |
| Binance public streams | Live crypto trades and book | Highest crypto spot volume; usually leads price discovery | Free |
| data.binance.vision | Crypto history: trades, aggTrades, klines | Bulk files with `.CHECKSUM` files; years of history | Free |
| Coinbase WebSocket | Live crypto trades and L2 book | Sequence numbers; the same venue as C1 | Free |
| IBKR market data | Live equities and futures | Execution-grade, not an archive | Per-exchange subscriptions |
| Databento | US equities, CME futures (history from 2010), full order-book schemas | Exchange-native timestamps; normalised formats | Historical data pay-as-you-go per GB; CME Standard subscription $199/month for new customers from 22 June 2026 |
| Massive (formerly Polygon) | US equities | Good coverage | About $29/month Starter *(secondary)* |

## Decision

1. **Equities, ETFs and futures execution: IBKR (E1).**
2. **Crypto spot execution: Coinbase (C1), conditional on confirming that Advanced Trade API access is enabled for Singapore accounts. If it is not, OKX SG (C2).** Independent Reserve (C4) is the tertiary fallback.
3. **Leveraged crypto exposure only through CME micro Bitcoin and Ether futures on IBKR**, never through offshore perpetuals.
4. **Research data:** Kraken WebSocket v2, Binance public streams and data.binance.vision for crypto (free, no account); Coinbase WebSocket recorded alongside; Databento pay-as-you-go for non-crypto history; IBKR for live non-crypto data.

**Rationale (against the drivers):**

- **IBKR** is the only option meeting D1–D3 that covers every non-crypto asset class in one account (D4), with the most mature API ecosystem (D7) and no API fee (D6). Its cost is that a funded account is needed before paper trading.
- **Coinbase** combines a licence (D1), a mature API (D7) and a full-depth feed that can also be recorded for research (D5). Its D2 status for Singapore accounts is unverified, so the decision is conditional and names the fallback in advance.
- **Keeping research data separate from execution** lets the system learn from the venues where crypto price discovery happens (Binance, Kraken) without holding an account there. It also enables lead–lag research: a signal from the leading venue, executed on the licensed one.
- **Databento's pay-as-you-go pricing** satisfies D6 for occasional historical pulls; a subscription is justified only above roughly 90 GB a month *(secondary estimate)*.

### Rollout by phase

| Phase | Data | Execution |
|---|---|---|
| 1. Data | Kraken WS v2, Binance streams, data.binance.vision, Coinbase WS | None |
| 2. Research | Add Databento pay-as-you-go (CME micro futures, US equities) | Open and fund the IBKR account |
| 3. Paper | Add IBKR live data, shared to the paper account | IBKR paper; Coinbase sandbox or OKX SG demo |
| 4. Live | Unchanged | IBKR for equities and futures; Coinbase (or OKX SG) for crypto spot |

## Consequences

- **Easier:** one broker integration covers every non-crypto asset class. The research data pipeline needs no accounts or credentials. Leveraged crypto exposure stays inside a regulated market.
- **Harder:** two execution integrations (IBKR, Coinbase) with different APIs, order models and failure modes, so a venue-neutral order interface is needed. Paper trading on IBKR needs a funded account and a data subscription shared to the paper user. Futures need MAS's knowledge assessment to be passed first.
- **Risks and mitigations:**
  - *Coinbase API unavailable for Singapore accounts:* verify before Phase 3; fallback OKX SG, then Independent Reserve.
  - *Lead–lag behaviour differs on the execution venue:* the trading signal reads Binance and Kraken but executes on Coinbase, whose lag profile may differ. Measure it on recorded data before building a strategy on it.
  - *Licence status changes:* most licence facts come from secondary sources. Confirm each venue on the MAS Financial Institutions Directory before depositing funds.
  - *Data-terms risk:* recording public feeds for private research is assumed acceptable. Re-read each venue's market-data terms before publishing any derived data.
- **Follow-up work:**
  - ADR 0002 defines the record schema and the `DataSource` / `Broker` interfaces. Because the scope spans asset classes and venues, records need an asset class and a venue in their key; that design belongs to ADR 0002.
  - Hosting region (Singapore or Tokyo, closer to the Asian venues) is an operations decision for the deployment milestone.

### Revisit triggers

- Kraken obtains an MAS licence → consider Kraken for crypto execution (its checksummed book is the best-instrumented feed).
- Alpaca receives a full CMS licence and offers individual accounts → reassess E4 against E1 on API ergonomics and cost.
- Coinbase API is unavailable or restricted for Singapore accounts → switch crypto execution to OKX SG and record the change in a superseding ADR.
- IBKR market-data or platform costs exceed the budget for two consecutive months → evaluate E3.
- MAS changes the retail DPT rules (for example on leverage) or the complex-products framework → reassess decision 3.
- Monthly Databento historical spend exceeds the subscription price → move to a subscription plan.

## Background

### How Singapore regulates the activities in scope

- **Securities and futures** are regulated under the Securities and Futures Act. A broker dealing in them for customers needs a **Capital Markets Services (CMS) licence**. IBKR, Saxo, moomoo and Tiger hold one; Alpaca has only in-principle approval.
- **Digital payment tokens** (cryptocurrencies) are regulated under the Payment Services Act 2019. A business that buys, sells or exchanges DPTs for customers needs a **Major Payment Institution (MPI)** licence with DPT services in scope (smaller businesses may hold a Standard Payment Institution licence). An **in-principle approval** is not a licence: it means MAS may grant one once conditions are met.
- **Retail DPT protections** under MAS guidelines: no financing, margin or leverage for retail customers, no trading incentives, and no public promotion. This is why licensed venues offer only spot trading to retail customers.
- **Digital Token Service Providers (DTSP)**: since 30 June 2025, Singapore-incorporated firms serving only overseas customers also need a licence under the Financial Services and Markets Act. This targets providers, not their customers, but it explains why several exchanges left Singapore.
- **MAS Investor Alert List**: a list of entities that may have been wrongly perceived as licensed by MAS. It is a warning, not a ban on use, but it rules a venue out as a counterparty for this project.
- **Complex products**: retail investors must pass a knowledge assessment before trading complex products such as futures. Historically this was the Customer Account Review (CAR) for listed products and the Customer Knowledge Assessment (CKA) for unlisted ones. In May 2026 MAS announced it will rename "Specified Investment Products" as "complex products" and merge the CAR and CKA into one assessment.

### Broker, exchange and data vendor

- An **exchange** runs the matching engine and order book (CME, SGX, Binance).
- A **broker** gives customers access to one or many exchanges, holds their assets and manages risk (IBKR). In crypto, the "exchange" usually plays both roles: it matches orders *and* holds customer assets, which is why licensing and counterparty risk matter so much.
- A **data vendor** redistributes exchange data in normalised form (Databento). Exchanges charge for their data, and vendors pass that cost on.

### Consolidated and non-consolidated US equity data

US equities trade on more than a dozen exchanges. The **consolidated feed** (the SIPs) combines every exchange's best quotes into the National Best Bid and Offer (NBBO). Brokers can give retail customers a cheaper **non-consolidated** feed from one exchange or a subset (for example Cboe One or IEX). It reflects only part of the market's liquidity, which is acceptable for research at minute scale and not for microstructure work.

### Paper trading and sandboxes

- A **paper account** simulates fills against live market data. It does not model queue position or market impact, so paper fills are optimistic.
- A **sandbox** is a separate environment, often with synthetic or static data, for testing that requests are well formed.
- Neither measures real execution quality. That needs small live trades, which is why the rollout keeps Phase 4 separate.

### Price discovery and lead–lag

When one asset trades on several venues, new information tends to appear first on the most liquid venue (**price discovery**), and other venues follow with a delay. In crypto spot, Binance usually leads. A **lead–lag** strategy reads the leading venue and trades on a lagging one before it catches up. That only works if the delay exceeds the system's own latency plus trading costs, which is why Umbra measures latency at every stage.

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
