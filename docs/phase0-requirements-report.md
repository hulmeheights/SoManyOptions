# Options Sniper: Phase 0 Requirements Report

**Date:** 2026-10-04
**Status:** Waiting for your answers (Section 8). No code gets written until you reply.

**Where the prices come from.** I researched everything on 2026-10-04. My environment's network policy blocked direct fetches of most provider pages, so the figures below come from web-search results, mostly limited to the provider's own domain. Check every price at checkout. Where sources disagree I say so and don't pick a winner. Anything I couldn't verify is marked **(unverified)**.

---

## 0. Summary

1. **You can't run this strategy on Webull UK as specified.** Webull UK's help centre says it doesn't support Level 3 options. UK customers get only long calls, long puts, long straddles, long strangles and protective puts. That rules out debit verticals and credit verticals at the account level, whether you use the app or the API. The API itself is also unclear. Webull's official OpenAPI agent-skills repo has a region table marking UK as ❌ for option trading, combo orders and option strategies. Webull UK's own OpenAPI marketing page says the UK API supports US options. **Until Webull UK support confirms otherwise in writing, assume Webull UK means single long options only, and maybe manual entry only.**
2. **Recommendation: execute through Interactive Brokers (UK).** IBKR supports debit spreads (options permission Level 2) and credit spreads (Level 3), combo orders through both of its APIs, and a paper account that mirrors live permissions and works with the API. Webull can stay behind the same `Broker` interface as a single-leg or manual mode if you want it.
3. **The PDT rule no longer constrains you.** The SEC approved FINRA's change in April 2026 (Regulatory Notice 26-10). It replaces the pattern-day-trader rule (the $25k minimum and 4-trades-in-5-days count) with an intraday margin standard, effective **4 June 2026**. Brokers may phase it in until **20 October 2027**. Webull UK states US options carry no PDT restriction. IBKR said it implemented the change on 4 June 2026. The plan is a configurable day-trade guard that is off by default, with no hard-coded hold-overnight rule.
4. **Running cost:** about **$145/month at the base case once live** ($105–$245 range) on the lean stack, plus **$599 one-off** for historical options data for Phase 1. Phase 1 needs only the one-off purchase. AI is $19–$143/month of the total. The GPT veto is the biggest AI cost.
5. **Minimum account size:** fixed and trading costs come to about **$2.2k/year**. Under about $22k of equity, that's more than 10% a year of cost drag before any edge. The system starts to be plausible at **about $25k–$50k**. The sizing rules set a separate floor: at 2% risk with a 1-contract minimum, narrow debit spreads need about $5k–$10k and single long options about $25k–$75k (Section 6).
6. **Two design points I'd challenge now** (details in Section 4.3):
   - A self-reported LLM "confidence ≥ 0.75" isn't a calibrated probability.
   - **20 paper trades can't validate anything statistically.** With n = 20 and a true 50% win rate, the 95% confidence interval on win rate is about ±22 percentage points. Keep the gate, but shadow-track every setup the system passes on so the AI layer can actually be evaluated.

---

## 1. Data

### 1.1 What the system needs

| Need | Used by | Freshness (live) | History for Phase 1 |
|---|---|---|---|
| Option chains: bid/ask, OI, volume, IV, greeks | universe, metrics, builder, exits | Real-time at order time. Delayed or EOD is fine for scans | **Yes.** Daily NBBO bid/ask back to 2007 if possible, to cover 2008, 2011, 2015, 2018, 2020 and 2022 |
| Underlying OHLCV (daily, plus intraday at order time) | MAs, ATR, realised vol, RS vs SPY, invalidation checks | EOD for signals, real-time for orders | Yes, adjusted for splits and dividends |
| Point-in-time market cap | universe (> $100B) | Daily | **Yes.** Using today's mega-caps in a backtest is survivorship bias. Rebuild the universe for each date |
| Earnings dates | setups (no earnings in hold window), exits (close before earnings) | Daily | **Yes.** Historical dates are needed for the filter |
| Ex-dividend dates | risk (early assignment on short legs) | Daily | Yes |
| News | news filter (Haiku), analyst context | Minutes to hours | **No.** You've ruled out backtesting the LLM layer on historical news |
| SEC filings (8-K, 10-Q/K, Form 4) | catalysts, analyst context | Hours | Optional. 8-K Item 2.02 filing dates are a free, point-in-time proxy for historical earnings dates |
| VIX or regime series | market regime filter | Daily | Yes |
| USD/GBP FX | sizing (live), tax (trade-date rate) | Live for sizing, daily for tax | Daily |
| NYSE holidays and early closes | scheduler | Static | n/a (Python library, no API) |

### 1.2 Provider options

**Live or near-live option chains**

| Provider | Plan / price per month | What you get | Notes |
|---|---|---|---|
| **Broker feed (IBKR)** | OPRA top-of-book **$1.50**, waived above $20/month in commissions. US equity real-time data is a separate fee **(price unverified)** | Real-time quotes through the API | Needed anyway for order pricing. We compute greeks and IV ourselves (py_vollib) from the quotes |
| Broker feed (Webull OpenAPI) | OpenAPI market data is bought **separately** from app subscriptions. US docs list OPRA non-display at **$2.99** non-pro | Real-time quotes | UK availability unclear (see Section 2) |
| **Massive** (formerly Polygon.io) | Options Starter **$29**, Developer **$79**, Advanced **$199** | Snapshots with greeks and IV on all three. Starter and Developer are **15-min delayed**, Advanced is real-time | Individual plans are non-professional only. Option quotes history starts only in **March 2022** (Advanced), so it's not enough for backtesting |
| ThetaData | Options Value **$40** (1-min, from 2020), Standard **$80** (tick, from 2016), Pro **$160** (tick, from 2012). Free tier: EOD from June 2023, 1-day delay | Real-time on paid tiers | Which greeks endpoints come with Value is **unverified** |
| ORATS | Delayed Data API **$199**, Live **$299**, Live Intraday **$599**, All-In **$899** | Chains with smoothed IV, greeks, IV rank, earnings | High quality, but expensive for 2–8 trades/month |

**Historical options data (Phase 1 backtests)**

| Provider | Price | Coverage | Verdict |
|---|---|---|---|
| **ORATS near-EOD** | **$599 one-off** backfill from 2007 (about 500 GB). Ongoing daily delivery $99/month. Individual licence | Snapshot taken **14 minutes before the close**: every strike and expiry on 5,000+ symbols, NBBO bid/ask, smoothed IV, full greeks | **Recommended.** Cheapest data that covers several regimes with real bid/ask. Download link is **valid for only 14 days** after purchase |
| ThetaData Standard / Pro | $80 / $160 per month for 1–2 months | Tick NBBO from 2016 / 2012 | Fallback. Misses 2008 and 2011, and tick data is more work |
| Massive Options Advanced | $199/month | Quotes from March 2022 only | Not enough history |
| ThetaData free | $0 | EOD from June 2023 | Not enough history |

ORATS also has an earnings-history endpoint in its Data API, returning `earnDate` and `anncTod` per ticker. Whether earnings dates come **inside the near-EOD files** is **unverified**. If they don't, EDGAR 8-K Item 2.02 dates are the free fallback.

**Everything else**

| Need | Recommended | Cost | Free-tier limits |
|---|---|---|---|
| Underlying daily bars | Massive Stocks Starter **$29** (15-min delayed, 5 years history). Stocks Basic is free (EOD, 2 years, 5 calls/min) | $0–29 | 5 calls/min on free. Enough for 43 tickers of EOD data, but slow |
| Market cap and fundamentals | Finnhub free (company profile). Point-in-time history from **SEC EDGAR** XBRL company facts (shares outstanding) × price | $0 | Finnhub: 60 calls/min |
| News | **Finnhub** free company news (1 year of history) | $0 | 60 calls/min. Paid tiers listed by third parties at about $12–$100/month **(unverified)** |
| Filings | **SEC EDGAR** (`data.sec.gov`) | $0 | Max **10 requests/s**. A User-Agent header with your name and email is **required** |
| Earnings calendar (live) | Finnhub free (forward calendar). Cross-check against ORATS or EDGAR | $0 | Free history is limited (about 1 month) |
| Ex-dividend dates | Massive reference data on stock plans, or ORATS **(confirm the exact endpoint during the data step)** | included | n/a |
| VIX | FRED (free API key) or Cboe's published history | $0 | n/a |
| FX, live (sizing) | Broker-reported balances and FX rate | $0 | n/a |
| FX, tax | **Bank of England** daily spot database. HMRC's exchange-rate API (monthly and spot, 2021 onward) as cross-check | $0 | n/a |
| Market calendar | `exchange_calendars` / `pandas_market_calendars` (library) | $0 | n/a |

### 1.3 Recommended stacks

**Lean starter stack (recommended):**
- **Phase 1:** ORATS near-EOD backfill ($599 one-off). Nothing else needs paying for during research.
- **Paper and live:**
  - Massive Options Starter + Stocks Starter ($58/month) for daily scans and 30-minute monitoring snapshots.
  - Broker real-time quotes for final validation and order pricing.
  - Finnhub, EDGAR, FRED and the BoE on free tiers.
- **Parity rule:** IV, IV rank and greeks are always computed **by our code from raw bid/ask**, in both backtest and live. Vendor-computed IVs are never mixed in. A different IV method between backtest and live is a common, silent reason live results drift from the backtest.

**Full stack (not recommended yet):**
- ORATS Live API ($299)
- ORATS near-EOD recurring ($99, for exact backtest parity)
- Massive Stocks Advanced ($199, real-time)

That's about $600/month in data alone. For 2–8 swing trades a month, real-time institutional data doesn't change decisions enough to justify it. Revisit only if forward paper results show the delayed data costing you fills or exits.

---

## 2. Execution

### 2.1 Webull UK findings

| Question | Finding | Confidence |
|---|---|---|
| Which option strategies can a UK account trade? | "Webull UK does not support Level 3 Options strategies – only long call, long put, long straddle, long strangle and protective put strategies are available." | High. Webull UK help centre (FAQ 1481, 1478) |
| Short options or spreads? | Not available (no Level 3) | High |
| Does the OpenAPI support UK? | Yes, through `developer.webull-uk.com`. **Minimum account size £1,000** | High |
| Does the UK API support **US options** orders? | **Sources conflict.** Webull UK's OpenAPI page says it "currently supports US stocks and US options". Webull's official `webull-openapi-skills` repo region table marks UK as ❌ for **Option Trading**, ❌ Combo Orders, ❌ Option Strategies, with order categories limited to `US_STOCK, US_ETF` | **Unresolved.** Ask Webull UK support (questions in Section 8) |
| Multi-leg orders by API? | Option strategies are **US region only** | High |
| Sandbox or paper by API? | A UAT (test) environment with shared test credentials, no application needed. The app's own paper trading covers US options | Medium (test environment documented on the US portal) |
| Auth | Token with 2FA: verified by SMS in the Webull app, valid **15 days** by default, must be refreshed before expiry | Medium (US developer docs) |
| Order rate limit | 600 requests/min for options orders | Medium (US docs) |
| Fees | **$0.50/contract** after the 90-day free promo. **FX conversion 0.50%** (Go plan) or 0.35% (Meridian) | High |
| PDT | Webull UK markets US options with "no PDT restriction" | High |

**What this means:** on Webull UK, the system can't place debit or credit verticals at all. It's limited to single long calls and puts. Long single options are the structure that **pays** the volatility risk premium rather than collecting it, and they need the largest account to size at 2% (Section 6). If you stay on Webull UK, Phase 1 will mostly be asking whether there's any edge in buying premium. Expect that answer to be "weak at best".

### 2.2 Broker options

| Option | Spreads? | Options by API? | Paper by API? | Verdict |
|---|---|---|---|---|
| **A. Webull UK** | ❌ | Unclear (see above). Multi-leg ❌ | UAT sandbox | Single-leg only, probably manual |
| **B. Interactive Brokers (UK)** | ✅ Level 2: long call/put spreads (debit verticals). Level 3: short call/put spreads (credit verticals) | ✅ TWS API and Web API both support combo/spread orders | ✅ Paper account mirrors live permissions and data. Same simulator as IBKR's own platforms | **Recommended** |
| C. tastytrade (through IG in the UK) | ✅ in the US product | tastytrade has a public API with multi-leg support | **(unverified for UK clients)** | Investigate only if IBKR is ruled out |

**IBKR costs (UK, US options):**
- Tiered commission of **$0.25–$0.65 per contract** depending on premium, with a **$1.00 minimum per order**, applied **per leg** on combos. Exchange and regulatory fees are extra.
- FX conversion costs **0.20 basis points, minimum $2**. Convert GBP to USD once in bulk and hold USD.

### 2.3 Setup steps (if you choose IBKR)

1. Open an IBKR UK individual account. Choose **margin** type: it's needed for credit spreads. Whether debit spreads are allowed in a cash account is **(unverified)**.
2. Apply for US options trading permission: **Level 2** for debit verticals, **Level 3** for credit verticals. Complete the knowledge and appropriateness questions.
3. Fund the account and convert to USD in one conversion.
4. Subscribe to OPRA top-of-book and US equity real-time data. Declare **non-professional** status; professional fees are far higher.
5. Request the **paper trading account**. It inherits your permissions and market data.
6. Install **IB Gateway** on the Mac Mini. Enable API socket access for localhost only. Expect regular re-authentication with 2FA (in practice about weekly). Monitoring alerts when the gateway is logged out.
7. Python library: the official `ibapi` or a maintained wrapper. That's an addition to your library list, so I'll confirm with you at the execution step.

### 2.4 Setup steps (if you stay on Webull UK)

1. Enable options in the app (appropriateness assessment). Keep the account at ≥ £1,000.
2. Build against the **UAT sandbox** with the shared test credentials.
3. Apply for production OpenAPI access at webull-uk.com/open-api. You receive an App Key and App Secret.
4. Buy an **OpenAPI** market-data subscription separately (app subscriptions don't carry over).
5. Create a token and verify it by SMS in the app. Re-verify before the 15-day expiry (monitoring sends an alert 3 days ahead).

### 2.5 Manual-approval fallback (built either way)

The fallback is used when the broker can't take the order by API: Webull UK options, an API outage, or if you simply prefer it.

1. **Ticket.** Once risk and sizing pass, the system creates an order ticket with an idempotency key (`uuid5(candidate_id + trade_date)`), legs, quantity and a limit-price ladder: mid, mid + ⅓ of the way to natural, mid + ⅔, then cancel.
2. **Telegram approval message.** Includes:
   - the ticket
   - structure maths (max gain, max loss, breakeven, reward-to-risk, net greeks)
   - both models' summaries
   - which rules matched
   - the risk-check results
   - an expiry time 30 minutes out

   It has Approve and Reject buttons. Presses after expiry, or repeat presses, are ignored (idempotent).
3. **On Approve:**
   - *API mode:* the system places the order and walks the ladder (about 10 minutes per step), then cancels if unfilled.
   - *Manual mode:* the system sends a **MANUAL TICKET** with exact field values to type into the Webull app, plus the ladder timings. You reply `/filled <id> <qty> <price>` or tap **Not filled**.
4. **Reconciliation, fail closed:**
   - A manual ticket counts as **open risk** in the risk layer from approval until confirmed filled, cancelled or expired. This prevents double entries and breaching limits.
   - If the broker API exposes positions read-only, the system cross-checks your confirmation. Any mismatch raises an alert and blocks new entries.
5. **Exits in manual mode:**
   - Same flow, but **time-critical exits** (before earnings, invalidation, kill switch) re-alert every 10 minutes until confirmed.
   - Exits never expire silently. An unconfirmed exit escalates.

### 2.6 What this means for the code

- The `Broker` interface reports capabilities: `multi_leg`, `api_option_orders`, `paper`, `option_quotes`.
- The builder only offers structures the active broker can execute.
- **Paper trading goes through our own simulator**, filling against live quotes with a conservative fill model. That simulator, not the broker's paper account, is the record of truth for the "20 trades and 3 months" live gate. Broker paper accounts are used for integration tests. They tend to fill too generously at mid.

---

## 3. Day trade rules

**Status of the rule.**
- FINRA amended Rule 4210 to remove pattern-day-trader designation: the 4-day-trades-in-5-days count, the $25,000 minimum equity and the 90-day freeze.
- In its place, brokers must monitor an **intraday margin deficit** on margin accounts.
- The SEC approved it in April 2026. It took effect **4 June 2026**, and firms may phase it in until **20 October 2027**. So some brokers can legally still run old PDT logic until they finish implementing.

**Your account.**
- Webull UK states US options trading has **no PDT restriction**.
- IBKR stated it implemented the new rules on **4 June 2026**.
- Either way, PDT doesn't apply to you.

**What still matters: settlement.** Options settle T+1. Cash-style accounts can restrict reusing unsettled proceeds. Whether Webull UK enforces US-style good-faith rules is **unverified**. **Sizing will always use broker-reported buying power and settled cash, never equity computed by our code.**

**Design:**
- `risk/day_trade_guard` takes `mode: off | count_limit | hold_overnight` and `max_day_trades_5d`. Default is `off`.
- It switches on automatically if the broker API reports a day-trade restriction, and sends an alert.
- When active:
  - Profit-target exits on the same day as entry are **deferred to the next session open** (logged).
  - Kill-switch and risk-halt exits are **always allowed**. A trading restriction is recoverable; an unmanaged position may not be.
  - Invalidation exits are evaluated on the daily close, so they never fall on the entry day anyway.
- At 2–8 trades a month, even the old 3-in-5 limit would rarely have bitten.

---

## 4. AI

### 4.1 Models by role and price per million tokens

| Role | Model | Input | Output | Cache read | Batch |
|---|---|---|---|---|---|
| News filter (materiality 0–10, ticker, direction, horizon) | `claude-haiku-4-5` | $1.00 | $5.00 | ~$0.10 | 50% off |
| Analyst (strict JSON decision) | `claude-opus-5-5` | $4.00 | $20.00 | $0.20 | 50% off |
| Veto (same input and schema, independent) | **GPT-6 Astra**: OpenAI's current flagship per its docs, released September 2026. Exact model ID to confirm in your OpenAI dashboard | $10.00 | $50.00 | $1.00 cached input ($12.50 cache write) | Batch/Flex 50% off |

Both providers support JSON-schema structured outputs. Reasoning tokens bill as output on both. On `claude-opus-5-5`, thinking can't be turned off; you control depth with the `effort` setting, which defaults to `medium`.

### 4.2 Monthly token cost at your target trade frequency

Assumptions:
- **News:** items are deduplicated and keyword-filtered before Haiku. Batched 10 per call, at about 800 system tokens per call plus about 400 input and 100 output tokens per item.
- **Analyst and veto:** about 20k input tokens per call (metrics, 2–3 candidates, news digest, rules) and about 8k output tokens including reasoning.
- **Calls:** one call each per confluence-gate pass, plus weekly re-reviews of up to 3 open positions (about 13/month).

| Scenario | News items/month | Gate passes/month | Haiku | Opus | GPT-6 Astra | **Total** |
|---|---|---|---|---|---|---|
| Low | 6,000 | 10 | $6 | $4 | $10 | **≈ $19** |
| **Base** | 15,000 | 30 | $15 | $10 | $26 | **≈ $51** |
| High (+20% retries) | 30,000 | 100 | $29 | $33 | $81 | **≈ $143** |

Per decision that's about $0.24 for Opus and $0.60 for GPT-6 Astra. Ways to cut the bill:
- Run the news filter through the Batch API (halves Haiku).
- Run the veto only on new entries, not weekly re-reviews (saves about 30% of the veto cost).
- Set spending caps in both consoles. I suggest $100/month each during build and paper.

### 4.3 Design issues to settle before the agents step

1. **Confidence isn't calibrated.** "Both ≥ 0.75" filters on self-reported numbers that don't map to real hit rates. Keep it as a conservative gate. Log every value, and after enough outcomes, plot a calibration curve before trusting the threshold.
2. **The two models aren't independent.** Both learned from overlapping public data, so their mistakes will be correlated. The veto adds less information than "two independent opinions" suggests. That's fine for a filter that can only say no.
3. **Shadow tracking (recommended addition).** For every gate pass, record the top candidate's hypothetical result under the same exit rules, **whether traded or not**. That's the only way to measure whether AI-approved setups actually beat AI-rejected ones, because forward data is the only clean data.
4. **Statistical power.** 20 trades gives about ±22 percentage points on win rate, and 100 trades about ±10. The 20-trade/3-month gate is a minimum before going live. It doesn't prove an edge, and the monthly report should say so.
5. **Fail closed.** Any of the following means **PASS**, logged:
   - a schema failure
   - a timeout
   - a refusal stop reason
   - disagreement between the models
   - missing input
6. **Untrusted input.** News text goes to the models as quoted data. Models have no tools and no order access. Their output is parsed against a schema, and the risk layer is code that no model can override.

---

## 5. Infrastructure

### 5.1 Mac Mini

**Runs:**
- the Python 3.12 service with APScheduler
- IB Gateway (if IBKR)
- the Telegram bot, using long polling so **no inbound ports are opened**
- a local market-data cache (SQLite/Parquet)
- a local **outbox** that queues journal writes when Supabase is unreachable. Trading must never depend on Supabase being up.

**Settings:**
- Prevent sleep, and restart automatically after a power failure (`pmset`).
- Run the service as a `launchd` agent with KeepAlive.
- Set macOS updates to manual.
- Use a wired network connection.
- A small UPS is a worthwhile one-off.

**Hours:** US regular session is 09:30–16:00 ET, normally 14:30–21:00 UK time. **Never hard-code the offset.** Use `zoneinfo` for America/New_York and Europe/London.
- Next mismatch: the UK falls back on **25 Oct 2026**, the US on **1 Nov 2026**. That week the open is 13:30 UK.
- In spring 2027 the US moves on **14 Mar** and the UK on **28 Mar**.
- Early closes and holidays come from the exchange calendar library.

**Jobs:**
- pre-market data and freshness check
- 30-minute exit checks during the session
- a near-close invalidation check
- after-close EOD scan and setups
- the daily status message
- the weekly re-review

### 5.2 Monitoring

- **Heartbeat:** the service writes a heartbeat row every 5 minutes.
- **Dead-man's switch.** A dead Mac Mini can't report itself, so a scheduled Supabase function (`pg_cron` plus an Edge Function) checks how old the last heartbeat is and messages Telegram if it's stale. That runs off the Mac.
- **Per-tick checks** before any entry, each failing closed:
  - quote age
  - broker session alive
  - token expiry (Webull) or gateway login (IBKR)
  - FX rate age
  - Supabase reachability (degrades to the outbox)
- On any failure, **new entries are blocked** and exits switch to alert-and-confirm.
- **Logs:** JSON lines with daily rotation and 30-day local retention. Nightly compressed backup (Time Machine plus a copy to Supabase Storage).

### 5.3 Supabase setup

- **Project:** EU region (London if offered). Use **Free while building and paper trading**. Move to **Pro ($25/month) before going live**: free projects pause after a week of inactivity, and Pro adds backups.
- **Tables:**
  - from your spec: `news_items`, `setups`, `candidates`, `decisions`, `orders`, `positions`, `daily_equity`
  - proposed additions: `heartbeats`, `model_calls` (tokens and cost per call), `fx_rates`, `tax_events`, `shadow_outcomes`, `config_snapshots` (which config produced each decision)
- **Security:**
  - RLS enabled on every table with no anon policies.
  - The service-role key lives only in `.env` on the Mac Mini.
  - Schema migrations are SQL files in the repo.

---

## 6. Total monthly cost and minimum account size

### 6.1 Monthly running cost (USD, lean stack)

| Item | Paper phase | Live |
|---|---|---|
| Massive Options Starter + Stocks Starter | $58 | $58 |
| Broker market data (IBKR OPRA $1.50, often waived; equity data price unverified) | ~$0–12 | ~$0–12 |
| Finnhub, EDGAR, FRED, BoE, Telegram | $0 | $0 |
| AI, base case (range $19–$143) | ~$51 | ~$51 |
| Supabase | $0 | $25 |
| Mac Mini electricity (rough) | ~$3–6 | ~$3–6 |
| **Total, base** | **≈ $120** | **≈ $145** (range ≈ $105–$245) |
| **One-off** | ORATS near-EOD backfill **$599** (Phase 1) | |

**Per-trade costs on top:**
- **IBKR vertical:** 2 legs × N contracts × up to $0.65, on entry and exit, so about **$2.60 × N per round trip** plus exchange and regulatory fees.
- **Webull UK single leg:** **$1.00 × N per round trip**, plus 0.50% on every GBP↔USD conversion. That's another reason to hold USD.
- The bigger hidden cost is **the bid/ask spread**. Phase 1 measures it with the mid-versus-natural fill assumptions.

**Full stack:** about $800/month (ORATS Live and recurring files, Massive real-time stocks, AI high case, Supabase Pro). Not justified at this trade frequency.

### 6.2 Minimum account size

**Floor 1: fixed costs.** Base live costs plus 5 round trips a month come to about **$2,220/year**.

| Equity | Annual cost drag |
|---|---|
| $10,000 | 22.2% |
| $25,000 | 8.9% |
| $50,000 | 4.4% |
| $100,000 | 2.2% |

**Floor 2: contract granularity.** At 2% risk you need at least one contract. Equity needed = max loss per contract ÷ 0.02, and it doubles to ÷ 0.01 after a 10% drawdown.

| Max loss per contract (example structure) | Min equity at 2% | At 1% (after drawdown) |
|---|---|---|
| $100 (e.g. $2.50-wide debit spread at 40% of width) | $5,000 | $10,000 |
| $200 (e.g. $5-wide debit spread at 40% of width) | $10,000 | $20,000 |
| $500 (single long option, $5.00 premium) | $25,000 | $50,000 |
| $1,000–$1,500 (single long option, $10–$15 premium) | $50,000–$75,000 | $100,000–$150,000 |

These premiums and widths are **illustrations, not quotes**. Real values come from the chains in Phase 1.

**Verdict:**
- On the lean stack with spreads (IBKR), the system can plausibly cover its costs from about **$25k–$50k**. At an illustrative 1.30 USD/GBP that's roughly £19k–£38k.
- Below about $20k, fixed costs alone are over 10% a year. Overcoming that would take an edge bigger than anything credibly documented for retail defined-risk options. Phase 1 will put real numbers on the edge.
- On **Webull UK (single long options only)**, the realistic floor rises to about **$50k+** because of contract size.
- "Covering costs" isn't the bar. The monthly report compares against simply holding SPY.

---

## 7. Checklist: accounts and keys you need to provide

Secrets go in `.env` on the Mac Mini (git-ignored). Nothing secret goes in `config.yaml` or the repo.

**Broker (one of these)**
- [ ] **IBKR UK** (recommended):
  - [ ] live account, margin type
  - [ ] US options permission Level 2, plus Level 3 if credit spreads survive Phase 1
  - [ ] paper account
  - [ ] OPRA and US equity data subscriptions, non-professional
  - [ ] IB Gateway installed
  - [ ] username for the gateway (no key; login with 2FA)
- [ ] **Webull UK** (only if chosen):
  - [ ] options enabled
  - [ ] OpenAPI production approval (≥ £1k)
  - [ ] `WEBULL_APP_KEY`, `WEBULL_APP_SECRET`, account ID
  - [ ] OpenAPI market-data subscription

**Data**
- [ ] ORATS account, plus the near-EOD backfill purchase (**download within 14 days**)
- [ ] `MASSIVE_API_KEY` (Options Starter and Stocks Starter, from paper phase onward)
- [ ] `FINNHUB_API_KEY` (free)
- [ ] `FRED_API_KEY` (free)
- [ ] SEC EDGAR User-Agent string (your name and a contact email; no key)

**AI**
- [ ] `ANTHROPIC_API_KEY` with a spending limit
- [ ] `OPENAI_API_KEY` with a usage limit. Confirm your account can call the flagship model; check for any organisation-verification requirement.

**Notifications and storage**
- [ ] `TELEGRAM_BOT_TOKEN` (from @BotFather), plus your numeric `TELEGRAM_CHAT_ID`. The bot will ignore every other chat.
- [ ] Supabase project: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` (server only), DB password for migrations

**Machine and admin**
- [ ] Mac Mini: Python 3.12, always-on settings, a dedicated macOS user for the service
- [ ] Confirm the trading account is a general (taxable) account, for the tax module
- [ ] Confirm you qualify as a **non-professional** market-data user

---

## 8. Decisions needed from you before Phase 1

1. **Broker:**
   - (a) IBKR UK for everything (recommended)
   - (b) Webull UK, single long options only, probably manual
   - (c) both: IBKR for spreads, Webull as a manual single-leg venue
2. **Webull UK check** (if you're keeping Webull in any form). Ask Webull UK support in writing:
   - "Does OpenAPI for UK accounts support placing US equity options orders? Which `instrument_type` values and order types are accepted?"
   - "Does it provide US option quotes and chains, and what does the OpenAPI OPRA subscription cost for UK users?"
   - "Is there any plan to offer Level 3 (spreads) to UK clients?"
   - Also confirm in the app that your options level shows long calls and puts only.
3. **Approximate account size range.** The code will always read live equity. I need a planning number to judge whether the lean stack and the 2% rule are viable for you.
4. **Phase 1 data purchase:** approve the ORATS near-EOD backfill ($599 one-off), or prefer ThetaData Standard for 1–2 months ($80/month, from 2016, no 2008/2011).
5. **Veto model:** confirm GPT-6 Astra (about half the AI bill), and whether the veto should also run on weekly re-reviews.
6. **Shadow tracking and the paper gate:** OK to add shadow outcomes? Do you want to keep 20 trades + 3 months as the minimum, or raise the trade count?
7. **SPY/QQQ/IWM.** UK retail clients generally can't buy US-domiciled ETF **shares** (PRIIPs/KID rules; the April 2026 move to the CCI regime hasn't simply removed that). Option chains on them may still be tradable. Please check in your broker. The system closes by 21 DTE, so it should never be exercised into ETF shares, but I want to know the chains are available before including them in Phase 1.

Once I have your answers, Phase 1 starts with the evidence review and then the backtests.

---

## Sources

Retrieved 2026-10-04. Most were seen through search-result extracts because direct fetches were blocked by network policy.

- Webull UK, Level 3 options FAQ: https://www.webull-uk.com/help/faq/1481
- Webull UK, option strategies FAQ: https://www.webull-uk.com/help/faq/1478
- Webull UK, OpenAPI page: https://www.webull-uk.com/open-api
- Webull UK, pricing: https://www.webull-uk.com/pricing
- Webull UK, US options page: https://www.webull-uk.com/trading-investing/options
- Webull UK developer docs: https://developer.webull-uk.com/apis/docs/
- Webull OpenAPI region table (official repo): https://github.com/webull-inc/webull-openapi-skills
- Webull OpenAPI options trading docs (US): https://developer.webull.com/apis/docs/trade-api/options/
- Webull OpenAPI FAQ (market data subscriptions): https://developer.webull.com/apis/docs/faq/
- Webull OpenAPI create token: https://developer.webull.com/apis/docs/reference/create-token/
- FINRA Regulatory Notice 26-10: https://www.finra.org/rules-guidance/notices/26-10
- SEC approval order, SR-FINRA-2025-017: https://www.sec.gov/files/rules/sro/finra/2026/34-105226.pdf
- FINRA investor insight, intraday margin: https://www.finra.org/investors/insights/intraday-margin-requirements
- Traders Magazine, regulators end PDT rule: https://www.tradersmagazine.com/featured_articles/regulators-end-pdt-rule/
- IBKR UK options commissions: https://www.interactivebrokers.co.uk/en/pricing/commissions-options.php
- IBKR combo orders (Web API): https://www.interactivebrokers.com/docs/web-api/trading/orders/orders-for-combos-spreads
- IBKR options trading permissions: https://www.ibkrguides.com/clientportal/optionstradingpermissions.htm
- IBKR paper trading account: https://www.interactivebrokers.com/campus/trading-lessons/request-paper-trading-account/
- IBKR market data pricing: https://www.interactivebrokers.com/en/pricing/market-data-pricing.php
- IBKR spot currency commissions: https://www.interactivebrokers.co.uk/en/pricing/commissions-spot-currencies.php
- Massive options: https://massive.com/options
- Massive option chain snapshot: https://massive.com/docs/rest/options/snapshots/option-chain-snapshot
- Massive options quotes: https://massive.com/docs/rest/options/trades-quotes/quotes
- Massive daily ticker summary (stocks): https://massive.com/docs/rest/stocks/aggregates/daily-ticker-summary
- ORATS near-EOD historical data: https://orats.com/near-eod-data
- ORATS Data API: https://orats.com/data-api
- ORATS historical data API: https://orats.com/docs/historical-data-api
- ThetaData pricing: https://www.thetadata.net/pricing
- ThetaData subscriptions: https://http-docs.thetadata.us/Articles/Getting-Started/Subscriptions.html
- Finnhub pricing: https://finnhub.io/pricing
- SEC, accessing EDGAR data: https://www.sec.gov/search-filings/edgar-search-assistance/accessing-edgar-data
- OpenAI pricing: https://developers.openai.com/api/docs/pricing
- OpenAI GPT-6 Astra model page: https://developers.openai.com/api/docs/models/gpt-6-astra
- Supabase pricing (third-party summary): https://makerkit.dev/blog/saas/supabase-pricing
- Bank of England exchange-rate database: https://www.bankofengland.co.uk/boeapps/database/
- HMRC exchange rates API: https://developer.service.hmrc.gov.uk/api-documentation/docs/api/xml/Exchange%20rates%20from%20HMRC
- GOV.UK, Capital Gains Tax rates and allowances: https://www.gov.uk/guidance/capital-gains-tax-rates-and-allowances
- tastytrade newsroom: https://tastytrade.com/newsroom/news/open-api-mcp-brings-ai-to-your-tastytrade-account/
- Edale, US ETFs and the CCI regime (third party): https://edale.co/us-uk-etf-kid-cci-reporting/

Anthropic model IDs and prices come from Anthropic's current API reference (pricing cached 2026-09-25).
