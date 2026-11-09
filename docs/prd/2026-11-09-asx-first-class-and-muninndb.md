# PRD — ASX as a First-Class Market, plus MuninnDB Agent Memory

| | |
|---|---|
| **Status** | Draft for review |
| **Date** | 2026-11-09 |
| **Repo** | `~/Documents/src/TradingAgents` |
| **Author** | DAIV (drafted from live graph + live run evidence) |
| **Decision requested** | Approve the market-abstraction architecture in §4 and the phase plan in §9 |

---

## 1. Executive summary

TradingAgents is a US-market application with multi-market *symbol* support bolted onto the edges. That distinction matters: `BHP.AX` already prices correctly and already benchmarks against `^AXJO`, but every guarantee the system is built around — point-in-time filings, insider disclosure, macro regime, sentiment corpus, plain-English prompt guidance — is silently American.

The instruction is that **ASX becomes first class**, which rules out the cheap path of adding a `.AX` branch inside each vendor. First class means the market is a declared dimension of the run, resolved once and consumed everywhere, with US behaviour preserved exactly as it is today.

Two epics ship together because they touch the same seam: the agents' understanding of *what they are looking at* (market context) and *what they have seen before* (memory).

---

## 2. Problem statement

### 2.1 The failure mode is silent degradation, not visible breakage

An ASX run today does not crash. It returns a confident report built on:

- no filed statements (SEC EDGAR refuses non-US filers),
- no insider data (reported as "none", which is a false absence),
- US monetary policy presented as the macro backdrop,
- US retail sentiment presented as the sentiment corpus,
- prompts that instruct the model to reason in SEC/Fed/S&P terms.

A user reading that report has no way to distinguish "this company has no insider selling" from "this venue is not covered". That is the core defect, and it is worse than an error because it produces trust rather than a bug report.

### 2.2 Verified gap inventory

Evidence below is from the live repository at `main`, gathered via Gortex graph queries and direct reads. Every row is a `file:line` fact, not an inference.

#### A. Already correct — do not rebuild

| Location | Status |
|---|---|
| `dataflows/symbols.py:103` `normalize_symbol` | `.AX` passes through intact (final passthrough rule) |
| `default_config.py:199` | `".AX": "^AXJO"` already in `benchmark_map` |
| `memory/settlement.py:16-38` `resolve_benchmark` | suffix match resolves `.AX` to ASX 200 correctly |
| `dataflows/vendors/yahoo/fundamentals.py:21` `_MAIN_UNIT` | AUD is a main unit, passes through |
| `portfolio.py:26-33` `PortfolioContext.currency` | free caller-supplied label; AUD works |
| `README.md:226` | already documents `.AX` |

#### B. Structurally US-only — the real work

| Location | Current assumption | ASX consequence |
|---|---|---|
| `default_config.py:161` `fundamental_data: "sec_edgar,yfinance"` | SEC EDGAR primary | ASX falls through to Yahoo, losing the filed-as-of guarantee |
| `dataflows/vendors/sec_edgar.py:195` | `NoMarketDataError(..., "not a US SEC filer")` | every ASX statement request spends a round-trip to fail |
| `dataflows/date_window.py:136` `withhold_live_profile` | "SEC EDGAR serves US filers as filed" | historical ASX run gets **no fundamentals at all** |
| `dataflows/date_window.py:175` `withhold_undated_statements` | same | same |
| `default_config.py:164` `macro_data: "fred"` | FRED | US monetary policy only |
| `dataflows/vendors/fred.py:44` `MACRO_SERIES` | 100% US series (`FEDFUNDS`, `DGS*`, `CPIAUCSL`, `PCEPILFE`) | no AU alias exists, so the tool cannot even be asked |
| `default_config.py:165` `prediction_markets: "polymarket"` | US event markets | no AU coverage to promise |
| `agents/analysts/sentiment_analyst.py:8, :142` | r/wallstreetbets, r/stocks, r/investing as ground truth | US retail psychology asserted for an ASX name |
| `dataflows/vendors/yahoo/fundamentals.py:148` `get_insider_transactions` | Form 4 (US) | returns "No insider transactions" — **false absence** |

#### C. Prompt text that steers the model to the US

| Location | Text |
|---|---|
| `agents/analysts/news_analyst.py:29` | names FRED, `fed_funds_rate`, `10y_treasury`, `yield_curve`, examples "Fed rate cut" |
| `agents/tools.py:236-251` `get_macro_indicators` | docstring aliases US + euro area only |
| `agents/tools.py:262-270` `get_prediction_markets` | examples "Fed rate cut", "US election" |
| `agents/context.py:131` | suffix examples `.TO`, `.L`, `.HK`, `.T`, `-USD` — `.AX` absent |
| `agents/context.py:105-128` `build_instrument_context` | injects `Exchange:` but never the quote currency |
| `cli/prompts.py:20, :66, :77` | US examples; empty ticker defaults to `SPY` |

#### D. Defects found while analysing

| Location | Defect | Severity |
|---|---|---|
| `agents/schemas.py::_coerce_optional_float` | cleans with `lstrip("$€£¥")`; `A$189.5` leaves `A`, float fails, returns `None`. **`entry_price` and `stop_loss` silently become `None`** | High — silent data corruption |
| `dataflows/symbols.py` `_ALIASES` | covers SPX500/NAS100/GER40/UK100/JP225/HK50, no `AUS200` | Low |
| `dataflows/date_window.py:42` `get_current_date` | `date.today()` on host clock; Perth evening is ahead of both NY and ASX dates | Medium — affects "is this run historical" |
| repo-wide | no exchange calendar; session boundaries inferred from returned bars | Note, not bug |

#### E. MuninnDB

| Location | Current state |
|---|---|
| `memory/log.py:11` `TradingMemoryLog` | append-only local log; `store_decision`, `get_past_context`, `update_with_outcome` |
| `memory/reflection.py:6` `Reflector` | LLM reflection on a settled decision |
| `memory/settlement.py:16-185` | benchmark-relative outcome settlement |
| **Gap** | no semantic recall, no supersession, no entity graph, no per-instrument scoping, no cross-run memory beyond one ticker's own log |

---

## 2.3 External data reality — what ASX can actually get

Verified 2026-11-09. This section decides Phase 2's shape.

| Source | Verdict | Consequence |
|---|---|---|
| **yfinance** | `.AX` **fully supported**: prices, `.info`, `.financials`, `.balance_sheet`, dividends, quoted in AUD. Suffix is mandatory and Yahoo treats bare `BHP` as a different security. Some minor listings have incomplete fundamentals. | ASX prices, technicals and news are a config change, not a build |
| **Alpha Vantage** | ASX support is a **grey area**. Not officially listed among supported exchanges; fundamentals coverage for Australian listings uncertain. | Do not depend on it for AU. Treat as unsupported until proven on a real ticker |
| **SEC EDGAR** | Structurally US-only. | Needs a replacement, not a patch |
| **ASX filings** | The licensed route is **ASX Online Information Services** (MIA REST API, auth-token flow, subscription). Announcements including Appendix 3Y director-interest notices are served through it. Third-party scrapers (e.g. Apify ASX announcements) exist but carry licensing and reliability risk. | Dated AU fundamentals require a paid subscription. There is no free official API |
| **FRED (AU series)** | OECD-sourced Australian interest-rate series do exist (e.g. `IRSTCI01AUM156N`), **but the OECD discontinued its MEI series in 2023**, so many of these stopped updating. The authoritative source is the RBA directly (table F1.1 cash rate target). | Do not wire AU macro to FRED. Either integrate RBA or declare macro unavailable for AU |

**The blunt version:** AU prices are cheap; AU fundamentals are a licence purchase; AU macro is a scraper or a gap. That is a budget decision, not an engineering one.

---

## 3. Goals and non-goals

### Goals

1. A market is a declared, resolved property of a run, not a scattering of suffix checks.
2. ASX and US are peers. Neither is "the default with exceptions".
3. US behaviour is bit-for-bit unchanged for existing users.
4. Every ASX data gap is **visible** — the report says what was unavailable and why.
5. Agents carry memory across runs via MuninnDB, scoped so no instrument's context leaks into another's.

### Non-goals

- Adding exchanges beyond US and AU in this increment (the abstraction must make it cheap, but we ship two).
- Replacing `TradingMemoryLog` outright. It stays as the audit record; MuninnDB is added as the recall layer.
- Intraday, options, futures, or FX trading logic.
- Building a licensed ASX data feed. We integrate what is legally available and label the rest.
- Backfilling historical ASX fundamentals before a legal dated source exists.

---

## 4. Architecture — the market abstraction

The decision this PRD asks you to ratify. One descriptor, resolved once, consumed everywhere.

```python
@dataclass(frozen=True)
class MarketProfile:
    id: str                      # "US" | "AU"
    name: str
    suffixes: tuple[str, ...]    # ("",) for US; (".AX",) for AU
    benchmark: str               # "SPY" | "^AXJO"
    currency: str                # "USD" | "AUD"
    calendar: str                # calendar id, or "infer-from-bars"
    vendor_chains: dict[str, str]
    macro_series: dict[str, str] # semantic name -> market series id
    macro_aliases: dict[str, str]
    sentiment_venues: tuple[str, ...]
    prompt_regime: str           # paragraph injected into agent context
    filing_source: str           # "sec_edgar" | "asx_announcements" | "none"
```

**Resolution rule.** `resolve_market(ticker, config)` matches the normalised symbol's suffix against each profile, longest suffix first, falling back to the configured default profile. `BHP.AX` → AU. `BHP` → US (today's behaviour, preserved). The result travels in run config and is readable by every node.

**Three consumption points, no more:**

1. **Vendor selection** — `data_vendors` becomes `profile.vendor_chains`, so `fundamental_data` resolves per market rather than globally.
2. **Context injection** — `build_instrument_context` gains `Market:`, `Currency:` and `Regime:` lines, so the model is told AUD and RBA rather than left to assume USD and Fed.
3. **Prompt regime** — one market-owned paragraph substituted into analyst prompts instead of hard-coded US institution names.

**Why this and not per-vendor `.AX` branches.** A suffix check inside `sec_edgar.py`, another inside `fred.py`, another inside the sentiment analyst gives three places to forget when a third market arrives, and no way to answer "what does this run actually believe about its market?". One descriptor answers that question once.

---

## 5. Requirements

Numbered as ideal-state criteria: each states what must be true and how it is falsified.

### 5.1 Market resolution

| # | Requirement | Falsified by |
|---|---|---|
| R1 | `resolve_market("BHP.AX")` returns the AU profile; `resolve_market("AAPL")` returns US | a test showing either resolving to the other |
| R2 | An unknown suffix (`.XX`) resolves to the default profile **and logs a warning naming the suffix** | a run where an unknown suffix passes silently |
| R3 | `benchmark_map` and `MarketProfile.benchmark` cannot disagree; one is derived from the other | a test where `benchmark_map` is edited and AU benchmark does not follow |
| R4 | Market resolution is recorded in `run_settings` and visible in the saved report | a saved ASX report that does not name AU |

### 5.2 Data vendors

| # | Requirement | Falsified by |
|---|---|---|
| R5 | A vendor declares which markets it supports; the router never offers an unsupported vendor | `sec_edgar` being attempted for a `.AX` symbol (observable in logs) |
| R6 | For AU, `fundamental_data` never routes to `sec_edgar` | a request log showing an ASX statement call to EDGAR |
| R7 | Unavailable data is reported as **unavailable for this market**, never as "none reported" | `get_insider_transactions("BHP.AX")` returning a no-data string |
| R8 | An AU historical run either gets dated statements or states plainly that it has none | a report asserting fundamentals without stating the source date |
| R9 | `macro_data` for AU resolves to an AU series or fails loudly | an AU run returning `FEDFUNDS` |

### 5.3 Currency and numeric parsing

| # | Requirement | Falsified by |
|---|---|---|
| R10 | `_coerce_optional_float` parses `A$189.5`, `AU$189.5`, `US$189.5`, `$189.5`, `189.5` | a unit test where any returns `None` |
| R11 | An unparseable price raises or logs at ERROR; it never silently becomes `None` | a run with `entry_price=None` and no error |
| R12 | Instrument context carries the quote currency | an ASX report whose agents were never told AUD |

### 5.4 Prompts and sentiment

| # | Requirement | Falsified by |
|---|---|---|
| R13 | No analyst prompt names a US institution for an AU run | `grep` of the rendered AU prompt for "Fed", "SEC", "S&P" |
| R14 | Sentiment venues come from the market profile | `r/wallstreetbets` appearing in an AU run's system message |
| R15 | `.AX` appears in every suffix example list the model is shown | `agents/context.py:131` unchanged |

### 5.5 MuninnDB memory

| # | Requirement | Falsified by |
|---|---|---|
| R16 | Each run writes a decision memory to MuninnDB with instrument, date, rating, and outcome pending | no memory present after a run |
| R17 | Agents recall prior theses for **this instrument only**; no cross-instrument leakage | an AAPL run showing `BHP.AX` memories |
| R18 | A settled decision supersedes rather than duplicates its own prior version | two active memories for the same run |
| R19 | Recall failures degrade to today's behaviour and are logged; they never block a run | a run aborting because MuninnDB is down |
| R20 | `TradingMemoryLog` remains the audit record; MuninnDB is recall, never the sole copy | a run whose decision exists only in MuninnDB |

### 5.6 Regression

| # | Requirement | Falsified by |
|---|---|---|
| R21 | Every existing US test passes unchanged | any test modified to accommodate the abstraction |
| R22 | A full US run produces the same decision path as before the change | a diff in node sequence or decision schema for the same inputs |

---

## 6. MuninnDB integration design

Grounded in live probe of the local daemon (`v0.9.0`) plus repo and documentation reads, 2026-11-09.

### 6.1 Shape

| Option | Verdict |
|---|---|
| Bind `muninn_*` as tools the agents call | **Rejected on three counts:** the graph is synchronous (`graph.invoke`) while LangChain's MCP adapter is async-only; the live surface is ~40 tools, which MuninnDB's own agent evaluation calls hostile and recommends cutting to 8–12; and it hands the poisoning vector to the model |
| Direct client from Python | **Partly viable, with a sharp edge:** REST on 8475 is plain JSON and sync-friendly, but is a materially narrower surface than MCP — no `as_of`, no `tags_all`/`tags_any`, no `relevance_band`, and `EvolveRequest` carries only `{new_content, reason, vault}` |
| **Thin adapter behind `tradingagents/memory/`** | **Chosen.** Preserve the existing interface and add a Muninn back-end beside the markdown log. Everything valuable here (valid time, supersession, tag scoping) is a write/read *policy*, not an agent capability |

**Transport decision that follows:** time-travel and tag-filtered recall exist only on MCP (`http://127.0.0.1:8750/mcp`, streamable HTTP, no MCP host required). If Phase 3 needs `as_of`, it talks MCP; if it needs only writes and tag-filtered reads, REST on 8475 is simpler. Ports: 8474 MBP, 8475 REST, 8476 UI, 8477 gRPC, 8750 MCP.

### 6.2 The adapter

```python
class MarketMemory:                      # new, beside TradingMemoryLog
    def remember_decision(run) -> str    # op_id=..., upsert_mode=True
    def recall_past(ticker, as_of) -> str
    def settle(memory_id, outcome) -> None   # evolve, then update_tags
    def recall_similar(decision, k) -> list
```

Touchpoints: `memory/log.py` (interface), `graph/trading_graph.py:309-371` (`_memory_step`, `record_decision`), `default_config.py:86-90` (new `muninn_*` keys), `backtest.py:170` (per-run vault).

LangGraph's checkpointer is short-term thread state; MuninnDB is cross-run memory. Nothing here touches the checkpointer.

### 6.3 Scoping — corrected from the first draft

**One vault per environment, isolation by enforced tag filter.** Not one vault per market, and not one per ticker.

- Vaults: `tradingagents`, `tradingagents-prod`. Backtests get `tradingagents-bt-<run_id>`, deleted at end, so hindsight lessons never reach live memory
- Every write carries `ticker:<SYM>` and `market:<AU|US>`; every recall passes `tags_all=["ticker:<SYM>", "market:<AU|US>"]`

Why not per-market vaults: **one API key binds to exactly one vault**, so splitting by market means N keys to rotate for no isolation the tag filter does not already give. Why not per-ticker: the app deliberately draws cross-ticker lessons (`n_cross=3` in `get_past_context`), and portfolio-level memory (analyst track record, risk appetite) belongs to no ticker.

The ticker and market must come from graph state, never from model output. Verified that `tags_all` is a hard filter, not a ranking hint.

### 6.4 What gets written

| Memory | `type` | Tags | Written by |
|---|---|---|---|
| Run decision | `decision` | `ticker:`, `market:`, `kind:decision`, `date:` | `record_decision()` |
| Same decision, settled | evolve the above | add `outcome:win\|loss\|flat` via `update_tags` | settlement |
| Reflection / lesson | `observation` | `ticker:`, `kind:lesson`, `window:` | `Reflector` |
| Standing thesis | `fact` (one live per ticker) | `ticker:`, `kind:thesis` | research manager |
| Analyst track record | `fact` (evolve in place) | `analyst:`, `scope:global` | settlement |
| Risk appetite | `preference` | `scope:portfolio` | manual, principal-owned |

**Write rules that are not optional:**

- Use `op_id` + `upsert_mode=true`, keyed deterministically (`tradingagents:decision:AAPL:2026-11-09`), so re-runs evolve rather than accrete rivals
- Use `evolve`, never a second `remember`, when a belief changes. A second `remember` leaves two live competing memories, and this is a repeated-write pipeline
- `upsert_mode` evolve inherits tags, so adding `outcome:` at settlement needs `muninn_update_tags` afterwards
- Pass `read_only:true` on any read whose result the app does not endorse; ordinary recall reinforces what it returns

### 6.5 Failure posture

**Gate on tags and dates, never on `score`.** `score` is max-normalised per query, so the top row of an unanswerable query still sits near 1. The intended quality signal is `relevance_band`, but on this install every probed row returned `uncalibrated` with basis `no_model_baseline`, so **the band is unusable here**. Design the filter around tags and the `resolved ≤ as_of` date gate the app already enforces at `log.py:93-107`.

Daemon unreachable or vault locked → log once, proceed exactly as today. Memory improves a run and must never be able to prevent one. Only settled outcomes are recallable; a pending decision must never enter `past_context`. Never let a run read its own fresh writes, since a missing embedding can score zero rather than unknown.

Net effect: the markdown log stays the system of record for settlement state; MuninnDB is a derived semantic index over lessons that have already settled.

---

## 7. Risks

| # | Risk | Mitigation |
|---|---|---|
| 1 | **Structured-output budget, not model fit.** The Portfolio Manager's structured call fell back to free text in the live run. A controlled probe disproved the model-fit theory: `Xaiomi/mimo-v2.6-pro` **passes** the real `PortfolioDecision` schema on a 154,883-character prompt at `max_tokens=16384` and **fails** on the identical prompt at `max_tokens=4096`. A reasoning model spends its budget thinking, and a tool call needs room after the thinking. Four other models passed the short-prompt probe. | Treat `max_tokens` as a hard requirement for structured agents, not a preference; add a schema-conformance test that fails on fallback; revisit whether 16,384 is enough for the real PM prompt |
| 2 | **No legal dated ASX filings source** at acceptable cost | Phase the AU historical guarantee behind a vendor decision (§9, Phase 3) |
| 3 | Abstraction creep: `MarketProfile` becomes a god object | Cap it at the nine fields in §4; anything else goes in config |
| 4 | Memory poisoning by an earlier wrong decision | Relevance-band gate; reflections carry the settled outcome so wrong theses are contradicted, not reinforced |
| 5 | US regression while adding AU | R21/R22 are gates, not goals |

---

## 8. Open questions for the principal

1. **ASX filings source — a spend decision.** There is no free official API. The licensed route is ASX Online Information Services (MIA REST, token auth, subscription) and it is the only path that gives dated statements and Appendix 3Y notices. Choose: fund the subscription, accept a third-party scraper and its licensing exposure, or ship AU with prices and news only and state plainly that dated fundamentals are unavailable for this market.
2. **AU macro — integrate RBA, or declare a gap.** FRED's Australian series are OECD-sourced and largely stopped updating in 2023, so they are not usable as a regime signal. Either wire the RBA's F1.1 cash-rate table directly, or mark `macro_data` as unavailable for AU and let the prompt say so.
3. **Structured-output token budget.** With `max_tokens=16384` the real PM prompt still fell back once. Do you want the budget raised (say 32,768) for structured agents, or a schema-conformance retry that raises the budget and retries once before falling back to free text?

---

## 9. Phases

| Phase | Content | Exit condition |
|---|---|---|
| **0 — Truth** | Fix the `A$` parse bug; make venue-unavailable distinct from no-data; add `.AX` to examples; `AUS200` alias | R7, R10, R11, R15 green |
| **1 — Abstraction** | `MarketProfile`, `resolve_market`, vendor support declarations, prompt regime substitution | R1-R6, R13, R14, R21, R22 green; US tests untouched |
| **2 — ASX data** | AU vendor chain, insider handling, macro handling, calendar decision | R8, R9 green; a real ASX run produces a report |
| **3 — MuninnDB** | Adapter, vault/tag scheme, decision + thesis + reflection writes | R16-R20 green |
| **4 — Proof** | One full ASX run and one full US run, both with evidence | side-by-side report |

---

## 10. Acceptance

The increment is done when a single command produces an ASX report that:

1. names the market and currency in its own text,
2. cites the filing source **and its date**, or states that none exists for this market,
3. never presents an unavailable data channel as a negative finding,
4. shows no US institution names unless the instrument is US-listed,
5. is reproducible from the same inputs, and
6. leaves every US run behaving exactly as before.

---

## Appendix — evidence provenance

- Gap inventory: Gortex graph queries plus direct reads, 2026-11-09, repo at `main`.
- Vendor chains and FRED series map: `default_config.py:158-168`, `fred.py:44+`.
- Live end-to-end run: NVDA, trade date 2026-11-07, decision `Underweight`, 453.9s, via CCR at `http://127.0.0.1:3456/v1`.
- Model-fit defect: `Portfolio Manager: structured-output invocation failed (structured output returned no parsed result); retrying once as free text`, from that run's log.
- Structured-output probe (2026-11-09): all five deep-tier candidates (`mimo-v2.6-pro`, `glm-5.3`, `deepseek-v4-pro`, `claude-sonnet-5.5`, `gpt-6.1-sol`) passed the short-prompt schema probe with `rating=Hold`. The budget test then isolated the cause: `mimo-v2.6-pro` passed at `max_tokens=16384` and failed at `4096` on an identical 154,883-character prompt. The defect is budget, not model.
- Currency defect: `agents/schemas.py::_coerce_optional_float`, read directly.
