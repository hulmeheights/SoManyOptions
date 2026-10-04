# Crypto Swing: Phase 0 Requirements Report

**Date:** 2026-10-04
**Status:** Waiting for your answers (Section 8). No code gets written until you reply.

**Where the facts come from.** I researched everything on 2026-10-04. This environment's network policy blocks direct fetches of exchange and provider pages, so every figure below comes from web-search extracts. I limited searches to the provider's own domain wherever I could. Check every price at checkout. Anything I couldn't confirm from a primary source is marked **(unverified)**, together with how we'll confirm it. All prices are USD unless stated, because that's how the providers bill.

---

## 0. Summary

1. **Crypto.com Exchange works for this design, with four problems to engineer around.**
   - **Fees are high.** At Level 1 (under $10k of 30-day volume), spot costs **0.25% maker and 0.50% taker**. A round trip with a maker entry and a taker stop exit is **0.75% before spread and slippage**. Every backtest will use these numbers.
   - **There is no retail sandbox.** The UAT environment is by invitation, for institutional accounts only. We build a simulated paper engine against live Crypto.com order books.
   - **API keys with trading enabled must be IP-whitelisted.** Your Mac Mini needs a **static public IP**. Most UK home broadband doesn't provide one by default.
   - **Spot orders lock balance.** You can't rest a full-size stop and three take-profit orders on the same coins at once. The answer is one **OCO (take-profit plus stop) per tranche**, which Crypto.com supports (Section 2.2).
2. **Isolate the bot from your Litecoin.** A trade-only key on your main account can sell your LTC. Run the bot in a **dedicated Exchange sub-account** and treat that sub-account's equity as "the account". Separately, HMRC pools LTC **per person, not per account**. The tax module therefore needs your personal LTC transactions as well, or the same-day and 30-day matching will come out wrong (Section 3.4).
3. **Phase 1 can be done for $129 one-off, or $429 with unlock history.**
   - **Survivorship-free universe:** one month of CoinGecko Analyst ($129). It keeps history for inactive and delisted coins.
   - **Hourly OHLCV with real exchange volume** for volume profiles: Binance's free public data archive.
   - **Funding and open-interest history:** the same Binance archive, also free.
   - **Optional:** one month of the DefiLlama API ($300) for token-unlock history.
   - **Not backtestable at a sane price:** on-chain data across 200 alts, social data and news. These become forward-only layers.
4. **Lean running cost: about $130/month while paper trading and about $160/month live** (range about $100–$275). AI is $31–$180 of that. The full stack is about **$1,600/month**, or about **$2,600 with Glassnode**.
5. **Minimum account size.** On the lean stack, the system starts to make sense at **about $75k** in the bot sub-account and is comfortable from about $150k. At $25k–$75k it can run, but fixed costs eat a large share of any plausible edge. Below about $25k they swamp it. **The full stack only makes sense at about $750k or more.** My recommendation is to pay for a data layer only once Phase 1 or paper trading shows that layer adds something.
6. **Venue risk.** The FCA's new cryptoasset regime started taking applications on 30 Sep 2026 and comes into force on **25 Oct 2027**. Firms that aren't authorised by then can't keep serving UK clients. Crypto.com is FCA-registered today (Foris DAX UK Ltd). Its authorisation under the new regime is not yet known. We keep the exchange behind an interface so it can be swapped.
7. **Three design points I'd challenge now:**
   - **15 paper trades can't validate an edge.** At a true 50% win rate, the 95% interval on win rate is about ±25 points. Keep the gate as a safety minimum, and shadow-track every gate pass so the AI layer can be evaluated.
   - **The spec has no risk-per-trade number.** Sizing by stop distance needs one. I propose 1% of sub-account equity.
   - **The confluence gate is eight hard ANDs,** two of which (catalyst, on-chain) can't be backtested. Expect few trades, so the "2 months + 15 trades" gate may take longer than 2 months to meet.

---

## 1. Data

### 1.1 What the system needs

| Module | Data | Resolution | History needed |
|---|---|---|---|
| universe | Market cap rank, circulating and total supply, stablecoin/wrapped flags, Crypto.com instrument list, 24h volume, order book | Daily (book: on demand) | Point-in-time ranks back to 2018 (backtest) |
| timeframes, metrics | OHLCV | 1W, 1D, 4h (and 1h for volume profiles) | 200+ days for MAs, 52 weeks for distance from high, about 1 year of 1h/4h bars for volume profiles |
| metrics (relative strength) | BTC and universe OHLCV | Daily | Same as above |
| metrics (derivatives context) | Perp funding, open interest, liquidations | 1h–8h | 2020 onward (perps for most alts didn't exist earlier) |
| metrics (on-chain) | Active addresses, exchange net flows, holder concentration | Daily | 1 year or more |
| metrics (fundamentals) | Fees/revenue, TVL | Daily | 1 year or more |
| metrics (supply) | Unlock events, next 60 days | Event | Forward schedule. Past events too, for the backtest |
| metrics (dev) | GitHub commits and contributors | Weekly | 1 year |
| news | Headlines with asset tags | Minutes | Forward only (no LLM backtest, per your rule) |
| social | Mention volume and sentiment vs baseline | Hourly–daily | About 90-day baseline, forward |
| sizing, execution | Balances, open orders, fills, fee tier | Real time | — |
| tax | GBP value of every fill | Per fill | — |

### 1.2 Provider options

**Market data and prices**

| Provider | Purpose here | Free tier | Paid | Notes |
|---|---|---|---|---|
| **Crypto.com Exchange API** | Live candles, tickers, order book for the venue we trade on | Free with any account | — | `public/get-candlestick` timeframes run from 1m to 1M, including 1h, 4h, 1D and 7D. Public market data is limited to 100 requests/s per method per IP. **How far back candle history goes is unverified,** and so is the maximum `count` per call. I'll test it from your Mac at the start of the data step. Can't cover 2018. |
| **CoinGecko** | Point-in-time market caps, ranks and supply; **inactive/delisted coins** | Demo: 10k credits/month, 1 year of history | Basic $35/month (2 years). **Analyst $129/month** ($103 billed yearly): daily data from 2013, hourly from 2018, and history for inactive coins. Lite $499/month | Paid plans only: `/coins/list?status=inactive` and history for delisted coins. That's what makes a survivorship-free universe possible. Volume is aggregated across venues, including low-quality ones, so it's fine for ranks but not for volume profiles. |
| CoinMarketCap | Alternative to CoinGecko | Basic: 15k credits, 1 year daily, 1 month intraday | Builder $29 (3 years), Startup $79 (all-time history), Growth $299, Professional $699 | **Which plan unlocks point-in-time historical listings (rank snapshots) is unverified.** I recommend CoinGecko because its handling of inactive coins is documented. |
| CoinDesk Data (ex-CryptoCompare) | Exchange-level OHLCV | **Free tier retired in May 2026** | Prices only via sales | Personal plan: 365 days daily and 7 days hourly. Start-Up: 30 days hourly. Full history is Enterprise only. **Not recommended.** |
| **Binance public data archive** (`data.binance.vision`) | Free bulk hourly spot klines for the backtest, with real per-venue volume | Free, no key | — | Monthly and daily zip files for all symbols. **Whether delisted symbols' files are retained is unverified.** I'll check by listing the archive in Phase 1. It only covers coins Binance listed, so CoinGecko fills the gaps at lower quality. |

**Derivatives context (read-only)**

| Provider | Free | Paid | Notes |
|---|---|---|---|
| **Binance USD-M public data** | Funding-rate history (REST and monthly archives). Open interest via daily "metrics" archives. REST open-interest history only reaches back 30 days | — | Free history for the backtest. **The archive's start dates per symbol are unverified.** Live REST access from a UK IP is **unverified**, so I'll test it from your Mac. No key needed. |
| **Coinglass** | — | **Hobbyist $29/month** (30 requests/min, 80+ endpoints, personal use), Startup $79, Standard $299, Professional $699 | Aggregated funding, open interest and liquidations across exchanges. Cleaner than single-venue data. Hobbyist's history-depth and interval limits are **unverified**. |

**On-chain**

| Provider | Free | Paid | Notes |
|---|---|---|---|
| Glassnode | Studio only | **API needs Professional, from $999/month.** Experimental pay-per-call: $0.05 per metric call in USDC | Strongest on BTC/ETH, thinner on alts. **Too expensive** for this account-size range. |
| CryptoQuant | Basic: 10k credits, 30 days | **Professional $99/month:** on-chain API, 1 year of history, 500k credits, 120 requests/min. Premium $799/month: full history | Best value if Phase 1 shows a BTC/ETH exchange-flow regime filter helps. Alt coverage is limited. |
| Santiment | 1,000 calls/month. Restricted metrics have 1 year of history and a **30-day lag** | Sanbase Pro $49 (5k calls, still lagged). **Sanbase Max $249** (80k calls, 2 years, no lag). Business plans via sales | Broadest coverage of ERC-20 alts (flows, active addresses, social, dev activity). **The 30-day lag makes Free and Pro useless for live signals.** Which metrics are restricted needs checking per metric. |

**Fundamentals**

| Provider | Free | Paid | Notes |
|---|---|---|---|
| **DefiLlama** | **Open API: TVL, fees and revenue, prices** | API plan $300/month ($3,000/year): 1M calls, includes **unlocks** | The free API covers the fundamentals we need. |
| Token Terminal | Free web tier | Pro $325–$350/month (sources disagree). **API is custom-priced** | Skip. |
| Artemis | Free "Intern" web tier | API only on the top plan, via sales | Skip. |

**Supply (unlocks)**

| Provider | Paid | Notes |
|---|---|---|
| **Tokenomist** | **Pro $69/month:** 1,000 API calls/month, supply data from 1 year back to 1 year ahead. Standard API $249.95 (50k calls, ±2 years). Elite API $449.95 (500k calls, ±3 years) | 500+ tokens. Pro is enough for a weekly refresh of about 150 coins. |
| DefiLlama API plan | $300/month | Includes unlock schedules. One month in Phase 1 can pull past unlock events for the backtest. **Whether past events are included in full is unverified.** |

**Developer activity:** the GitHub REST API is free. A personal token gives 5,000 requests/hour, which is plenty for weekly polling of about 150 projects' repos. Repo-to-coin mapping comes from CoinGecko metadata. Honest note: I know of no good evidence that dev activity predicts **one-month** returns. Expect it to end up as context for the analyst, not a gate.

**News**

| Source | Cost | Notes |
|---|---|---|
| **RSS: CoinDesk, The Block, Decrypt** | Free | `https://www.coindesk.com/arc/outboundfeeds/rss/`, `https://www.theblock.co/rss.xml`, `https://decrypt.co/feed`. I'll check each URL on first fetch. |
| **CryptoPanic** | Developer plan free (**limits unverified**). Growth $199/month or $50/week (only 3,000 requests/month). Enterprise from $899/month | Server-side cache of about 30 seconds. Start on the free plan. Growth is poor value at 3,000 requests. |
| Messari | **Enterprise only**: Lite and Pro have been retired | Skip. |
| Crypto.com Exchange announcements (listings and delistings) | Free | Matters for delisting risk on open positions. How we'd ingest them (feed or page) is **unverified**. |

**Social**

| Provider | Cost | Manipulation risk and honest view |
|---|---|---|
| X API | **Pay-per-use: $0.005 per post read** ($0.001 for "owned" reads). Legacy Basic $200/month, Pro $5,000/month | 150 coins × 100 posts/day ≈ **$2,250/month**. Crypto X is heavily botted and full of paid promotion, and raw post counts are trivially gamed. **Not worth it.** |
| LunarCrush | Free "Discover" tier. **Individual $90/month** (limited endpoints, 10 requests/min, 2,000/day). Builder $300, Scale $900 | Pre-aggregated, with some bot filtering. Usable as a crowding and attention flag. Its history is thin, so it can't be backtested properly. |
| Santiment social | Inside Sanbase Max $249 (real time) | Social volume, plus weighted sentiment. Same lag problem below Max. |

**My view on social:** treat it only as a crowding and pump flag ("attention spiked 5× baseline while price stalled"). That's how your spec uses it, and any reasonable proxy does that job. **Defer paying for it until paper trading is running.** The interface ships with a null provider, so the gate treats social as "unknown" rather than "bullish".

### 1.3 Phase 1 backtest data plan (survivorship bias and its limits)

- **Universe membership:** CoinGecko Analyst for one month ($129).
  - Use `/coins/list?status=inactive` plus `/coins/{id}/market_chart/range` to rebuild **daily top-200 by market cap from 2018**, including coins that died or were delisted. That's roughly 1,000–1,500 unique coins and well under 100k credits.
  - Exclude stablecoins and wrapped tokens with a maintained category list. Keep it in the repo.
- **Prices and volume for setups and volume profiles:** Binance hourly klines from the free archive, which have real per-venue volume. Coins that never listed on Binance fall back to CoinGecko hourly price plus aggregate volume, flagged lower quality. Volume profiles for those coins are then skipped, not faked.
- **Fees and spreads:** Crypto.com Level 1 fees (0.25% maker, 0.50% taker) by default, with a sensitivity run at Level 2 and 3. Spread and slippage come from a model by liquidity bucket. **Start collecting hourly Crypto.com order-book snapshots now.** It's free, and by the end of Phase 1 we'll have real data to calibrate slippage instead of guessing.
- **What can't be reconstructed historically, stated plainly:**
  1. Which coins were **listed on Crypto.com and available to UK users** at each date. The backtest uses "top 200 plus a liquidity filter". The live universe adds the Crypto.com and UK filter.
  2. Crypto.com price history before the Exchange existed. The 2018 bear runs on Binance and CoinGecko data.
  3. **Perps for most alts didn't exist in 2018–2019.** The derivatives filter can only be tested from about 2020, and I'll report it separately.
  4. **On-chain across 200 alts:** not affordable with point-in-time integrity. Providers revise flow data when they relabel addresses, which leaks future knowledge. I'll test, at most, a **BTC/ETH-only** exchange-flow regime filter, and only if you buy CryptoQuant Professional for a month.
  5. **Unlocks:** without DefiLlama (one month, $300) the unlock filter can only be checked on recent data. With it, I'll test 2021 onward, where coverage exists.
  6. **News, catalysts and social:** forward-only (your rule 6 for news; data limits for social).

### 1.4 Recommended stacks

**Lean starter stack.** Paid layers are added only after Phase 1 or paper trading earns them.

| Need | Source | $/month |
|---|---|---|
| Live prices, candles, books, execution | Crypto.com Exchange API | 0 |
| Ranks, market caps, supply | CoinGecko Demo | 0 |
| Funding and open interest | Binance public REST (fallback: Coinglass Hobbyist $29 if Binance is unreachable from the UK) | 0 (29) |
| Fundamentals | DefiLlama free API | 0 |
| Dev activity | GitHub API | 0 |
| Unlocks (a hard gate in your spec) | **Tokenomist Pro** | 69 |
| News | RSS plus CryptoPanic free plan | 0 |
| Social | None at first (null provider) | 0 |
| On-chain | None at first | 0 |
| **Data total** | | **$69 ($98 with Coinglass)** |

**Full stack.**
- CoinGecko Analyst $129
- Coinglass Startup $79
- CryptoQuant Professional $99
- Santiment Sanbase Max $249
- Tokenomist Standard API $249.95
- DefiLlama API $300
- CryptoPanic Growth $199
- LunarCrush Individual $90

That's **about $1,395/month in data**. Glassnode Professional (from $999) would take it to about $2,400. I'd still exclude X, Token Terminal, Artemis and Messari.

---

## 2. Execution

### 2.1 Crypto.com Exchange: confirmed facts

| Item | Finding |
|---|---|
| API | Exchange API v1. REST requests are signed with HMAC-SHA256 using your key and secret, and there are WebSocket feeds for market and user data. New docs are at `exchange-developer.crypto.com`. Crypto.com also publishes an official CLI (`crypto-com/cdcx-cli`) that covers OCO and OTOCO. That's a useful reference for exact method names. |
| Key permissions | New keys are **"Can Read" only**. Trading must be enabled explicitly, and so must withdrawal: leave **withdrawal off**. **Enabling trading requires an IP whitelist.** |
| Sandbox | UAT is at `uat-api.3ona.co`, **by invitation for institutional accounts only**. Assume we can't use it, which means a simulated paper engine (Section 2.4). |
| Spot fees (30-day volume tiers) | L1 (<$10k): 0.25% / 0.50%. L2 (≥$10k): 0.20% / 0.40%. L3 (≥$50k): 0.15% / 0.25%. L4 (≥$250k): 0.10% / 0.20%. L5 (≥$500k): 0.08% / 0.18% (maker / taker). With a CRO balance, maker drops to 0% and L1 taker to 0.44%. The CRO amount needed should be checked on the fee page. Tiers recalculate daily at 04:00 UTC. |
| Order types | LIMIT and MARKET. Trigger orders: **STOP_LOSS** (fires a market order), STOP_LIMIT, TAKE_PROFIT and TAKE_PROFIT_LIMIT. Order groups: **OCO**, **OTO** (spot only) and **OTOCO** (entry plus stop plus take-profit). Stop-loss is supported for fully funded spot trades. Triggers can reference last, mark or index price. **Exact v1 method names and parameters are unverified.** I'll confirm them against live docs in the execution step. |
| Rate limits (confirmed) | `private/create-order`: 15 requests per 100ms per key. `public/get-candlestick`: 100 requests/s per IP. Other private limits are **unverified**. |
| Sub-accounts | API methods exist (`private/get-subaccount-balances`, `private/create-subaccount-transfer`). **Whether a key can be scoped to a single sub-account is unverified.** That's the first thing to check in the UI (Section 8). |
| Idempotency | `client_oid` on create-order. Uniqueness and length rules are **unverified**. Our idempotency key is generated locally and stored before sending, regardless. |
| ccxt | Has a `cryptocom` class with `triggerPrice` and `stopLossPrice`. **Whether it supports OCO and OTOCO is unverified.** |
| UK funding | GBP deposits via Faster Payments only (£20 minimum). GBP/USDT and GBP/USDC pairs exist. Quote currencies include USD, USDT, BTC, ETH, CRO and PYUSD. |

### 2.2 Problems and the design answer

1. **Spot balance locking vs. stop-plus-ladder.** On spot, coins committed to a resting sell order can't back a second sell order. A full-size stop and three take-profit orders can't coexist.
   **Design:** after the fills, place exchange orders per tranche:
   - one third: **OCO** with a target-1 limit sell and a stop
   - one third: **OCO** with a target-2 limit sell and a stop
   - final third: a plain **STOP_LOSS**, which software trails

   Every coin is then covered by a stop on the exchange at all times. A stop move (to breakeven, or a trail) means cancel and replace. That leaves a gap of milliseconds to seconds with no stop on that tranche. The engine logs it, retries, and alerts if it lasts more than about 10 seconds. The stop is never moved down; that rule is enforced in code before any replace is sent.
2. **Stop-market vs. stop-limit.** Use **STOP_LOSS (market)**. A stop-limit can be skipped entirely in a crash, which defeats "a crash is always covered". The cost is slippage on gaps. The backtest models that by filling stops at the worse of the stop level and the next bar's open.
3. **Daily-close logic vs. intraday hard stops.** Your spec evaluates exits on daily closes but also wants a hard stop on the exchange, which fires intraday. Phase 1 will test both:
   - (a) an intraday hard stop at the logical level plus 0.5×ATR
   - (b) a daily-close stop at that level, plus a wider catastrophic hard stop on the exchange

   Then I'll recommend one with evidence.
4. **Static IP.** If the Mac's public IP changes, every authenticated call fails. The system fails closed: no new orders, and an alert. Exchange-side stops still protect open positions. Software trailing and take-profit management stop working. Fixes, in order:
   1. a static IP from your ISP
   2. a small UK VPS with a static IP as the egress hop over WireGuard (about £5/month, **unverified**)
5. **Fees.** A position's round trip is roughly 0.6–1.2% all-in, from maker entries, mixed maker and taker exits, and spread. That's tolerable for a swing system targeting moves of 15% or more. It's **a real drag on the many scratch trades** that a breakeven stop produces.
   - Moving the stop to "breakeven" should mean entry plus round-trip fees, or it's actually a loss.
   - Entries and take-profits are always limit orders (maker). Only stops and emergency exits pay taker fees.

### 2.3 Account isolation (your LTC)

- Create an Exchange **sub-account** for the bot. Fund it with the capital you're allocating, and create the API key there.
- If keys can't be scoped to a sub-account, **move your LTC out of the Exchange account** (to the App or self-custody) before enabling any trade key. A trade-only key can't withdraw, but it **can sell**.
- **"The account" in all limits means the bot sub-account's equity,** fetched live before every decision and never hard-coded.
- The bot treats LTC like any other coin for signals. The correlation and risk check can **optionally include your external LTC holding**, so the bot doesn't stack more LTC risk onto a large existing position. That's your call (Section 8).

### 2.4 Paper engine (default mode)

- Uses live Crypto.com tickers and order books (`public/get-book`) and the same order state machine as live, behind one `Broker` interface. Paper and live then differ only in the adapter.
- **Fill model:**
  - Limit buys fill only when the market **trades through** the limit, not merely touches it.
  - Market and stop orders walk the live book for the order's size.
  - Fees come from your actual tier.
  - Trigger orders fire on the same reference price live would use.
- Paper keeps its own balances, seeded from a configured paper equity. It's journalled exactly like live, so the "2 months and 15 trades" gate is measured on real records.
- **Live requires:** `mode: live` in config, **plus** ≥ 60 days and ≥ 15 closed paper trades in the journal, **plus** a Telegram confirmation. The check runs in code at start-up and on every order.

### 2.5 ccxt or a native adapter

**Recommendation: a thin native adapter on `httpx`** for orders and account calls, with ccxt as an optional market-data helper. The API is small and signed JSON. Owning it gives us:
- exact control of `client_oid` and retries
- OCO and OTOCO, which ccxt may not wrap
- precise error-code handling for fail-closed behaviour

Contract tests run against recorded responses.

### 2.6 Setup steps (when we reach the execution step)

1. **Before step 1:** UK onboarding requires an appropriateness quiz, and new users have a 24-hour cooling-off period. You've presumably done both already.
2. Create a sub-account and fund it from the main account.
3. In the sub-account, create an API key: **Can Read + Can Trade, withdrawal off, IP whitelist set to the static IP**.
4. A second **read-only** key for paper mode and monitoring, so paper never holds trade permissions.
5. Keep the secrets in `.env` on the Mac only (`chmod 600`). `.env` is already git-ignored.

---

## 3. Regulatory (UK)

### 3.1 Restrictions that affect us

- **Derivatives:** restricted for UK users on Crypto.com. This matches your rules. Derivatives data is read-only context from other venues.
- **On-chain perpetuals** (Onchain wallet): restricted for UK.
- **UK onboarding:** the appropriateness quiz and the 24-hour cooling-off period. Incentives are barred during cooling-off, and UK users are excluded from the Welcome Bonus.
- **No FSCS or FOS protection** for cryptoassets held at Crypto.com.

### 3.2 Privacy coins

| Asset | Finding | Status |
|---|---|---|
| XMR | A crypto.com price page shows it as "not tradable" | **Treat as untradable** |
| ZEC | Crypto.com has price pages and published ZEC market commentary in Sep/Oct 2026, but I found **no confirmation that it's tradable on the Exchange for UK users**. Kraken UK delisted XMR but kept ZEC | **Unverified. Please check in your account** |
| DASH and others | No UK-specific finding | **Unverified** |

- The UK has **no statutory privacy-coin ban**. From **July 2027**, EU rules bar regulated platforms from handling anonymity-enhancing tokens. How ZEC's optional privacy will be treated is undecided. None of this applies to the UK directly, but it may change Crypto.com's global listing decisions.
- **How the code handles it:**
  - The universe is built from `public/get-instruments` intersected with a maintained deny-list.
  - Any order rejected for jurisdiction reasons automatically adds the asset to the deny-list and alerts you.
- **The ZEC case study in Phase 1 is a backtest, so it runs either way.** Whether the live system could have traded it is a separate question I'll state explicitly.

### 3.3 Venue risk: the new FCA regime

- **Today:** Crypto.com UK (Foris DAX UK Ltd) has been **FCA-registered since 16 Aug 2022** for AML purposes, and holds EMI authorisation from March 2025.
- **New regime:** applications opened **30 Sep 2026** and close **28 Feb 2027**. The regime comes **into force on 25 Oct 2027**, and firms without authorisation can't continue serving UK clients.
- **Mitigation:**
  - The exchange stays behind the `Broker` interface.
  - Monitoring checks for a delisting or a termination notice.
  - If Crypto.com isn't authorised in time, we port to another venue. Kraken and Coinbase have UK entities. **Their authorisation status is also unknown.**

### 3.4 Tax implications that shape the build

- **HMRC share pooling applies per person and per asset, across all your accounts and wallets.** The bot's LTC trades and your personal LTC holding share one Section 104 pool and one set of same-day and 30-day matching rules. The tax module needs an **import for your external transactions in any asset the bot also trades** (LTC first), or its gains figures will be wrong.
- **Stablecoins.** Buying a coin with USDT is a disposal of USDT. Each one is a recordable event, though usually a tiny gain. If you quote in USD instead, those events don't happen; whether "USD" on the Exchange is fiat is **unverified**. That's an argument for USD-quoted pairs.
- **GBP value per fill.** Most pairs are USD or USDT, so every fill needs a GBP rate at the time of the trade.
  - Options: Crypto.com's GBP/USDT pair at the fill time, or the Bank of England daily rate.
  - HMRC accepts a reasonable, consistent method. We'll fix one in config and log the source of every rate.
- **Exchange reporting.** From **1 Jan 2026**, under the Cryptoasset Reporting Framework, UK platforms collect user and transaction data. The first reports, covering 2026, are due to HMRC by **31 May 2027**. HMRC will have Crypto.com's view of your trades, so our records need to reconcile with it.

---

## 4. AI

### 4.1 Models and prices (per million tokens)

| Role | Model | Input | Output | Cache read | Batch |
|---|---|---|---|---|---|
| News filter | `claude-haiku-4-5` | $1.00 | $5.00 | about $0.10 | 50% off |
| Analyst | `claude-opus-5-5` | $4.00 | $20.00 | $0.20 | 50% off |
| Veto | **GPT-6 Astra**, OpenAI's current flagship per its docs. Confirm the exact model ID in your OpenAI dashboard | $10.00 | $50.00 | $1.00 (unverified this session) | 50% off (unverified) |

- **Anthropic API details:**
  - Structured JSON output is enforced by schema on both providers.
  - On `claude-opus-5-5` thinking can't be turned off. Depth is set with `effort`, which defaults to `medium`. I'd run the analyst at `high` for entries and `medium` for re-reviews.
  - Refusals arrive as a stop reason, which we treat as **pass**. I'll enable the API's server-side refusal fallback.
- **A cheaper veto option:** OpenAI's GPT-6.1 Sol costs $2/$10, a fifth of Astra's price. Your spec says "flagship", so the estimates below use Astra.

### 4.2 Monthly token cost

Assumptions:
- **News:** about 100–1,000 items/day after dedupe and an asset-keyword prefilter. Batched 10 per call, at about 480 input and 100 output tokens per item, which is about $0.001 per item.
- **Entry decision:** about 20k input and 8k output tokens (including reasoning) per model.
  - Per decision that's **$0.24 for Opus and $0.60 for Astra**.
- **Re-review:** your spec asks for a daily re-review of each open position, and "either model flagging" means both run. That's about 8k input and 3k output tokens, or **$0.09 for Opus and $0.23 for Astra**.

| Scenario | Gate passes/month | Average open positions (daily re-reviews) | Haiku | Opus | GPT-6 Astra | **Total** |
|---|---|---|---|---|---|---|
| Low | 10 | 2 (60) | $3 | $8 | $20 | **≈ $31** |
| **Base** | 30 | 3 (90) | $9 | $16 | $39 | **≈ $63** |
| High (+20% retries) | 80 | 6 (180) | $30 | $43 | $107 | **≈ $180** |

**Re-reviews are most of the GPT cost.** A cheaper option that keeps your safety intent:
- Opus re-reviews daily.
- GPT re-reviews **weekly**, and also immediately when Opus flags a position, price comes within 1 ATR of the stop, or a material news item hits that asset.

That cuts the base case to about **$48**. Other levers:
- Prompt caching of the fixed rules block
- the Batch API for the news filter
- spending caps in both consoles: **$100/month each** during build and paper

### 4.3 Design issues to settle before the agents step

1. **Confidence isn't calibrated.** "Both ≥ 0.75" filters on self-reported numbers. Keep it as a conservative gate. Log every value, and plot a calibration curve once there are enough outcomes.
2. **The two models aren't independent.** They learned from overlapping public data, so their errors correlate. That's acceptable for a filter that can only say no.
3. **Shadow tracking (recommended addition).** For every gate pass, record the hypothetical result under the same exit rules, **whether traded or not**. Forward data is the only clean way to learn whether AI-approved setups beat AI-rejected ones.
4. **Statistical power.** At 15 trades the 95% interval on win rate is about ±25 points. At 100 trades it's about ±10. The live gate is a safety minimum and proves nothing. The monthly report will say so.
5. **Fail closed.** Any of the following means **PASS**, logged with the reason:
   - a schema failure
   - a timeout
   - a refusal
   - disagreement between the models
   - missing input
   - stale data
6. **Untrusted input.** News and social text go to the models as quoted data. The models have no tools and no order access. The risk layer is code, and no model output can change limits.

---

## 5. Infrastructure

### 5.1 Mac Mini

**Runs:**
- the Python 3.12 service (APScheduler, with `max_instances=1` and misfire grace set per job)
- the Telegram bot, using long polling so no inbound ports are opened
- the local data cache (Parquet and SQLite)
- a **local outbox** that queues journal writes when Supabase is unreachable. Trading never depends on Supabase being up.

**Settings:**
- Prevent sleep, and power back on automatically after an outage (`pmset`).
- Run the service under `launchd` with KeepAlive.
- Set macOS updates to manual.
- Use a wired network connection.
- A small UPS is a worthwhile one-off.

**Design principle:** every open position must be safe **with the Mac switched off**. That's why stops live on the exchange.

### 5.2 Schedule (24/7, everything in UTC)

Crypto never closes. All schedules run in UTC, and Telegram shows UK time. Crypto.com's daily candle boundary is assumed to be 00:00 UTC (**unverified**; I'll check on the first data pull).

| Job | When |
|---|---|
| Exit and stop checks, reconciliation of exchange orders vs. local state | Every 15 minutes |
| 4h entry timing within approved entry zones | Every 4h bar close |
| Daily close: metrics, signals, gate, AI calls, Telegram approvals | 00:10 UTC |
| Daily re-review of open positions, daily summary | After the daily-close job |
| Weekly regime and universe refresh | Monday 00:30 UTC |
| Data freshness checks (fail closed) | Before every decision |
| Order-book snapshots (slippage calibration) | Hourly |

### 5.3 Monitoring

- **External dead-man's switch: Healthchecks.io.** Its free plan has 20 checks and Telegram alerts. The service pings it every 5 minutes, and Healthchecks.io messages you if pings stop. That covers a dead Mac, a dead network or a crashed service, none of which can report themselves.
- **Per-cycle checks:** exchange connectivity, API key validity, data age per source, clock drift, disk space, Supabase reachability. Any failure blocks new entries and alerts you. Exits continue on exchange stops.
- **Telegram kill switch:**
  - `/halt` cancels pending entry orders and blocks new ones.
  - `/flatten` sells every bot position at market, after a confirm step.
  - Both are logged.
- **Logs:** JSON lines with daily rotation. A nightly compressed backup goes to Time Machine and to Supabase Storage.

### 5.4 Supabase setup

- **Project:** EU region. Use **Free while building and paper trading**. The free plan gives 500 MB, and projects pause after a week of inactivity. Move to **Pro ($25/month) before going live**, for backups and no pausing.
- **Tables:**
  - from your spec: `assets`, `signals`, `news`, `decisions`, `orders`, `positions`, `daily_account`
  - proposed additions: `fills`, `heartbeats`, `model_calls` (tokens and cost per call), `fx_rates`, `tax_events`, `external_transactions` (your personal LTC etc.), `shadow_outcomes`, `config_snapshots`
- **Security:**
  - RLS enabled on every table, with no anonymous policies.
  - The service-role key lives only in `.env` on the Mac.
  - Migrations are SQL files in the repo.

---

## 6. Total monthly cost and minimum account size

### 6.1 Phase 1 (one-off)

| Item | Cost |
|---|---|
| CoinGecko Analyst, 1 month (survivorship-free universe) | $129 |
| Binance public data archive | $0 |
| DefiLlama API, 1 month (unlock history), **optional** | $300 |
| CryptoQuant Professional, 1 month (BTC/ETH flow-regime test), **optional** | $99 |
| **Total** | **$129 (up to $528)** |

### 6.2 Lean running cost

| Item | Paper | Live |
|---|---|---|
| Data (Section 1.4) | $69 | $69 |
| AI (base; range $31–$180) | $63 | $63 |
| Supabase | $0 | $25 |
| Healthchecks.io, Telegram, GitHub, RSS | $0 | $0 |
| Static IP or VPS (**unverified**) | £0–£10 | £0–£10 |
| **Total (base)** | **≈ $132** | **≈ $157** |
| Range | ≈ $100–$250 | ≈ $125–$275 |

**Full stack:** about $1,395 for data, $180 for AI and $25 for Supabase, so **about $1,600/month**. Add Glassnode and it's **about $2,600/month**.

### 6.3 Minimum account size

This is fixed cost as a percentage of bot capital per year. Trading costs (about 0.6–1.2% per round trip on position size) come on top.

| Bot capital | Lean (about $1.9k/year) | Full (about $19k/year) |
|---|---|---|
| $20k | 9.4% | 96% |
| $50k | 3.8% | 38% |
| $100k | 1.9% | 19% |
| $250k | 0.8% | 7.7% |
| $500k | 0.4% | 3.8% |

**What edge can we expect?** We don't know until Phase 1. An illustrative assumption: the system averages about 30% exposure, since it's selective and capped at 50%. Earning 15–30% a year on deployed capital would then be **about 5–10% a year at the account level**, and that would already be a good result.

For the system to be worth running, fixed costs should be **no more than about a quarter of that**. That gives:
- **Lean stack:** sensible from **about $75k** (if the edge is at the top of that range), comfortable from about $150k. At $50k, costs would take about 40–75% of the assumed edge.
- **Full stack:** about **$750k or more**
- **Below about $25k:** fixed costs swamp any plausible edge

At that size, run paper only, or the lean stack with the GPT veto on entries only.

Phase 1 may also show the rules don't beat holding BTC on a risk-adjusted basis. If so, I'll tell you to stop. That would be cheaper than any of the numbers above.

---

## 7. Checklist: accounts and keys

**Needed for Phase 1:**
- [ ] **CoinGecko Analyst** API key (1 month). Optional: DefiLlama API (1 month) and CryptoQuant Professional (1 month).
- [ ] Confirmation that the Mac can reach `data.binance.vision` and Binance's public futures REST API. I'll give you a one-line test.

**Needed before paper trading:**
- [ ] **Crypto.com Exchange:** UK account with KYC and the appropriateness quiz done, plus a **bot sub-account**.
  - A read-only key for paper and monitoring.
  - Later, a **trade key: Can Read + Can Trade, withdrawal OFF, IP-whitelisted**.
- [ ] **Static public IP**, or a VPS egress you control.
- [ ] **Anthropic API key**, with a spend cap set in the Console.
- [ ] **OpenAI API key**, with a spend cap. Confirm the flagship model ID shown in your dashboard.
- [ ] **Telegram:** bot token from @BotFather and your chat ID.
- [ ] **Supabase:** project URL and service-role key.
- [ ] **CoinGecko Demo** key (free) for live ranks.
- [ ] **Tokenomist Pro** API key.
- [ ] **CryptoPanic** API token (free plan).
- [ ] **GitHub** fine-grained token with public-repo read access only.
- [ ] **Healthchecks.io** account (free) and its ping URL.

**Only if Phase 1 or paper trading justifies them:** Coinglass, CryptoQuant, Santiment Max, LunarCrush, DefiLlama API, CryptoPanic Growth.

**Not needed:** a Binance account (its public data needs no key) or an X API account.

---

## 8. Decisions needed from you before Phase 1

1. **Bot isolation.** Will you fund a dedicated Crypto.com Exchange sub-account for the bot? Please check one thing in the API key screen: can a key be created for a single sub-account? If it can't, will you move your LTC off the Exchange account?
2. **Your LTC in the risk layer.** Should the correlation and exposure check count your external LTC holding? I recommend yes, as information that blocks stacking, but never as something the bot can trade.
3. **Risk per trade.** I propose **1.0% of sub-account equity** lost if the stop is hit. Size is then risk ÷ stop distance, capped at 15%. For example: a 10% stop gives a 10% position, and a 25% stop gives 4%. Six positions all stopped would cost about 6% plus gap slippage.
4. **Budget.** Do you approve the Phase 1 one-off ($129 minimum)? Do you want the optional DefiLlama ($300) and CryptoQuant ($99) months? Do you approve the lean running stack (about $130–$160/month)?
5. **Static IP.** Does your broadband have one? If not, which do you prefer: an ISP static IP, or a small UK VPS hop?
6. **Your fee tier.** What 30-day volume tier and CRO balance does your account have? Do you want to hold CRO for fee discounts? My view is no, unless the saving clearly beats the CRO price risk.
7. **Base currency for the idle reserve (50% or more of the bot account).** Should it be held in USD, USDC/USDT or GBP? GBP avoids FX noise in your results. USD avoids a conversion on every trade.
8. **Privacy coins.** In your UK account, can you trade ZEC, XMR and DASH on the Exchange? Please send a screenshot or a yes/no for each.
9. **AI cost control.** Should the GPT veto re-review weekly, triggered by events (about $48/month base)? Or daily, as specified (about $63/month)?
10. **Universe volume rule.** Is "$20M average daily volume" **global** volume (CoinGecko), with Crypto.com liquidity checked separately by spread and depth? Or is it volume **on Crypto.com only**? I recommend global volume, plus a Crypto.com check:
    - spread ≤ 0.20%
    - depth within 1% of mid ≥ 5× the intended order size

    Crypto.com-only volume would shrink the universe sharply. I haven't measured by how much.

**Smaller spec points.** I'll use these defaults unless you object:
- **Breakeven** means entry plus round-trip fees.
- **"Two consecutive stop-outs"** counts losing stops only, not breakeven or trailing exits.
- **Exposure** is marked to market. The 50% cap blocks **new** entries and never forces sales. A position that grows past 20% of equity triggers an alert.
- **"No meaningful progress after 3 weeks"** will be defined numerically in Phase 1, for example a maximum gain of less than 1× the stop distance. It won't be tuned to any case study.
- **Unlocks:** a coin with no unlock data passes only if CoinGecko shows it ≥ 95% circulating (fully-circulating PoW coins like LTC and BTC). Otherwise the gate fails closed.

---

## Sources

Retrieved 2026-10-04 through search-result extracts, because direct page fetches were blocked by network policy.

**Crypto.com**
- Exchange fees and limits: https://crypto.com/exchange/document/fees-limits?tab=2
- Exchange API docs (v1): https://exchange-docs.crypto.com/exchange/v1/rest-ws/index-insto-8556ea5c-4dbb-44d4-beb0-20a4d31f63a7.html
- Exchange developer docs: https://exchange-developer.crypto.com/exchange/v1/docs/api/rest-introduction
- Common API reference: https://exchange-developer.crypto.com/exchange/v1/docs/api/rest-common-api-reference
- Help, API keys: https://help.crypto.com/en/articles/3511424-api
- Help, API key management: https://help.crypto.com/en/articles/13843786-api-key-management
- Help, sub-accounts: https://help.crypto.com/en/articles/5580404-sub-accounts
- Help, advanced order types: https://help.crypto.com/en/articles/4451025-advanced-order-types
- Help, OCO orders: https://help.crypto.com/en/articles/5807203-one-cancels-the-other-oco-orders
- Help, stop-loss orders: https://help.crypto.com/en/articles/9669060-stop-loss-orders
- Help, derivatives geo-restrictions: https://help.crypto.com/en/articles/4894470-derivatives-trading-geo-restrictions
- Help, Onchain perpetuals geo-restrictions: https://help.crypto.com/en/articles/9292190-crypto-com-onchain-perpetuals-geo-restrictions
- Help, UK onboarding quizzes: https://help.crypto.com/en/articles/10058644-uk-onboarding-quizzes-appropriateness-assessment-client-categorisation
- UK financial promotions rules: https://crypto.com/document/finprom-rules-uk
- Help, UK GBP deposits via FPS: https://help.crypto.com/en/articles/10145913-uk-retail-users-gbp-fiat-deposit-and-withdrawal-via-fps-exchange
- Help, OTC trading pairs: https://help.crypto.com/en/articles/10660416-crypto-com-exchange-otc-trading-pairs
- Help, Welcome Bonus: https://help.crypto.com/en/articles/9575514-welcome-bonus
- Crypto.com official CLI: https://github.com/crypto-com/cdcx-cli
- ZEC price page: https://crypto.com/us/price/zcash/usd
- XMR price page: https://crypto.com/us/price/monero

**Regulation and tax**
- FCA register, Foris DAX UK Ltd: https://register.fca.org.uk/s/firm?id=0014G00002antHVQAY
- Crypto.com EMI authorisation: https://crypto.com/en/company-news/crypto-com-receives-authorisation-as-an-electronic-money-institution-from-the-united-kingdoms-financial-conduct-authority
- FCA, how the gateway will operate: https://www.fca.org.uk/firms/new-regime-cryptoasset-regulation/how-gateway-will-operate
- FCA, opens the gateway: https://www.fca.org.uk/news/press-releases/fca-opens-gateway-regulated-crypto
- GOV.UK, collecting cryptoasset user and transaction data (CARF): https://www.gov.uk/guidance/collecting-cryptoasset-user-and-transaction-data
- GOV.UK, CARF implementation: https://www.gov.uk/government/publications/cryptoasset-reporting-framework/implementation-of-the-cryptoasset-reporting-framework-carf
- Privacy-coin context: https://www.ccn.com/education/crypto/countries-banning-privacy-coins-monero-zcash-2026/

**Market data**
- CoinGecko API pricing: https://www.coingecko.com/en/api/pricing
- CoinGecko, inactive and delisted coin history: https://support.coingecko.com/hc/en-us/articles/23190618031385-Can-we-access-historical-data-for-inactive-or-delisted-coins-via-CoinGecko-API
- CoinGecko, querying coin data: https://docs.coingecko.com/docs/querying-coin-data
- CoinMarketCap API pricing: https://coinmarketcap.com/api/pricing/
- CoinDesk Data pricing: https://developers.coindesk.com/pricing/
- CoinDesk Data, free tier change: https://data.coindesk.com/blogs/changes-to-coindesk-data-indices-api-free-tier-access
- Binance public data: https://github.com/binance/binance-public-data
- Binance open interest statistics: https://developers.binance.com/docs/derivatives/usds-margined-futures/market-data/rest-api/Open-Interest-Statistics
- Binance funding rate history: https://developers.binance.com/docs/derivatives/usds-margined-futures/market-data/rest-api/Get-Funding-Rate-History

**Derivatives, on-chain, fundamentals, unlocks**
- Coinglass pricing: https://www.coinglass.com/pricing
- Glassnode API setup: https://docs.glassnode.com/basic-api/api
- Glassnode x402 pay-per-call: https://docs.glassnode.com/basic-api/x402
- CryptoQuant APIs: https://cryptoquant.com/apis
- Santiment API plans: https://academy.santiment.net/products-and-plans/sanapi-plans/
- Santiment pricing: https://app.santiment.net/pricing
- DefiLlama pricing: https://docs.llama.fi/pro-api
- Token Terminal pricing: https://tokenterminal.com/pricing
- Artemis pricing: https://artemisanalytics.com/pricing
- Tokenomist pricing: https://tokenomist.ai/pricing

**Developer, news, social**
- GitHub REST API rate limits: https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api
- CryptoPanic API plans: https://cryptopanic.com/developers/api/plans
- CoinDesk RSS: https://www.coindesk.com/coindesk-news/2021/09/17/coindesk-rss
- Messari API tiers: https://docs.messari.io/api-reference/permissions
- Messari plan retirement FAQ: https://docs.messari.io/user-guides/welcome/plan-deprecation-faq
- X API pricing: https://docs.x.com/x-api/getting-started/pricing
- X API owned-reads pricing update: https://devcommunity.x.com/t/x-api-pricing-update-owned-reads-now-0-001-other-changes-effective-april-20-2026/263025
- LunarCrush pricing: https://lunarcrush.com/pricing

**AI and infrastructure**
- OpenAI pricing: https://developers.openai.com/api/docs/pricing
- Supabase pricing: https://supabase.com/pricing
- Healthchecks.io pricing: https://healthchecks.io/pricing/
- ccxt Crypto.com stop orders: https://github.com/ccxt/ccxt/pull/14799

Anthropic model IDs and prices come from Anthropic's current API reference (pricing cached 2026-09-25).
