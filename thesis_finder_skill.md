# Thesis Finder Skill — Amadeus Epsilon Fund #1

## Purpose

This skill evaluates companies from a fixed investment universe against a user-supplied macro investment thesis and returns a curated, ranked list of recommendations. Each recommendation includes a paragraph of qualitative rationale and the key fundamental data that supports it.

---

## Step 0 — Input Validation

Before doing any analysis, confirm that the user has provided a **macro investment thesis**: a statement describing a market-level view, rate environment, sector preference, or growth/income/risk posture.

- If the input is a company name, ticker, or question about a single stock → **reject it**. Reply: *"This skill requires a macro thesis as input, not a single-stock question. Please describe your overall market view or investment strategy and I will find stocks that fit it."*
- If the input is too vague to derive sector or factor tilts (e.g., "make money") → **ask one clarifying question** before proceeding.

---

## Step 1 — Parse the Thesis Into Screening Criteria

Extract the following from the thesis statement:

| Dimension | What to identify |
|---|---|
| **Rate environment view** | Higher-for-longer, falling, stable? |
| **Growth vs. income tilt** | Capital appreciation, dividends, or balanced? |
| **Preferred sectors** | Which sectors fit the thesis? |
| **Excluded sectors / factors** | What does the thesis explicitly avoid? |
| **Time horizon** | Short-term (weeks/months) vs. long-term? |
| **Risk posture** | Low, medium, or high tolerance for volatility? |

Output a brief summary of the derived criteria before proceeding. This is your analytical anchor — every subsequent decision must be traceable back to it.

---

## Step 2 — Universe Filtering

**You may only consider companies that appear in `universe.csv`.** Do not suggest, reference, or analyze any ticker or company outside this file. The universe contains approximately 101 companies across Nasdaq, NYSE, and TSX spanning 11 sectors.

### Hard Exclusions (remove before any analysis)

Remove the following categories unconditionally, regardless of valuation or news:

1. **Debt-heavy utilities and infrastructure** — Excludes: NEE, SO, DUK, FTS, EMA, AQN, ENB, TRP, PPL. These carry structural leverage that becomes a liability in a higher-for-longer rate environment. A large dividend yield does not offset the balance sheet risk.
2. **Volatile commodity-dependent materials** — Excludes: FCX, NTR, ABX, AEM, FNV, WPM, TECK.B, LIN (review LIN case-by-case as it has pricing power). Commodity-price exposure makes return forecasting unreliable.
3. **Low-end consumer discretionary** — Excludes: DOO, CTC.A, NKE (review case-by-case), MG. These names face headwinds from softening lower-end consumer spending and elevated borrowing costs on big-ticket or discretionary purchases.
4. **High-multiple, low-moat, low-revenue tech** — Excludes any tech name that cannot demonstrate: (a) significant and growing revenue base, (b) an established competitive moat (network effects, switching costs, or platform dominance), and (c) substantial cash reserves or free cash flow generation. Flag OTEX and CRM for close review.
5. **No single position over 25%** of the portfolio — Flag any recommendation that would naturally warrant outsized weighting.

### Portfolio-Level Concentration Limits

These limits govern final portfolio construction and override individual scoring results when breached. Apply them when building or rebalancing — they are not a one-time filter but a binding constraint on the output:

- **Maximum 35% per GICS sector** — no single sector may represent more than one-third of total portfolio weight. If a sector breaches this ceiling, rank picks within the sector by score and drop the lowest-scoring names first until the limit is met.
- **Maximum 50% combined (Technology + Communication Services)** — these two sectors are structurally correlated through shared exposure to AI, digital advertising, cloud, and consumer tech. Treat them as a single block for concentration purposes; the combined cap is non-negotiable regardless of how compelling the individual picks are.
- **Maximum 30% per investment sub-theme** — a "sub-theme" is a tightly linked cluster of holdings sharing a primary driver (e.g., AI infrastructure, digital payments, cloud software, GLP-1 pharma). Two companies that both rise and fall on the same catalyst are in the same sub-theme regardless of GICS sector. Cap combined weight in any one sub-theme at 30%.
- **Maximum 2 companies per GICS sub-industry** — more than 2 names in the same sub-industry (e.g., two payment processors, two hyperscaler-dependent semis) adds correlated risk without diversification benefit. If a third would be added, drop the lower-scoring name.

After exclusions and concentration limit review, compile your **Eligible Universe** — the subset of tickers eligible for full analysis.

---

## Step 2.5 — Portfolio Interaction Analysis

**Do not score individual stocks in isolation.** Before proceeding to Step 3, evaluate the proposed set of picks as a group. This step exists because individual scoring cannot detect portfolio-level failures: two highly-rated stocks can together produce a fatally concentrated portfolio.

### Sector Tally
Build a running sector-by-sector weight tally using the candidate picks and their preliminary suggested weights. Map each pick to its GICS sector. If any sector already hits 35% — or Tech + Comms combined hits 50% — before scoring is complete, flag it before adding further names from that sector.

### Sub-Theme Overlap
Identify which picks share a primary secular driver. Common sub-themes in this universe:
- **AI infrastructure** — semiconductors (NVDA, AMD, AVGO) and hyperscaler-adjacent platforms (MSFT, GOOGL, ORCL)
- **Digital payments** — payment networks (V, MA)
- **Cloud / enterprise software** — SaaS platforms (CRM, ORCL, MSFT, ADBE)
- **AI-linked pharma** — drug discovery / GLP-1 acceleration (LLY)
- **Discount / value retail** — consumer trade-down names (DOL, WMT, COST)

Compute combined sub-theme exposure. If any sub-theme would exceed 30%, rank picks within it by score and reduce or eliminate the weakest name.

### Correlation Flags
Flag any pair of picks where:
- Both are in the same GICS sector **and** the same sub-theme
- Both are rate-sensitive in the same direction (e.g., two high-multiple growth stocks that both sell off on rate-rise news)
- Combined, they represent a single-factor bet (e.g., two names that are effectively a pure AI capex proxy)

Document flagged pairs and note what happens to the portfolio if the shared driver reverses.

### Market Context Stress Check
Cross-reference the current pick list against the warnings in `market_context.md`:
- If Tech + Comms exposure exceeds 50% and the note flags momentum-driven rotation risk → this is a contradiction. Resolve it explicitly before proceeding.
- If rate-sensitive leverage is present and the note flags higher-for-longer risk → flag it.
- If consumer spending softening is noted and the portfolio has no defensive or trade-down exposure → flag the gap.

**Only proceed to Step 3 after completing this analysis.** The interaction analysis output is a prerequisite for scoring, not an afterthought.

---

## Step 3 — Live Data Pull via bigdata.com

For every company in the Eligible Universe, pull the following metrics live. Note explicitly if any data point is missing, stale, or appears unreliable — do not silently substitute estimates.

### Universal Metrics (all companies)
- Current price and 52-week range
- Market capitalization
- Revenue (TTM) and YoY revenue growth rate
- EBITDA margin
- Free cash flow (TTM)
- Net debt / EBITDA (or equivalent leverage ratio)
- P/E (TTM) and YoY Revenue Growth Rate *(Forward P/E not available from bigdata.com — use TTM P/E as the valuation anchor and pair it with YoY revenue growth to assess the growth-adjusted multiple)*
- Price/Free Cash Flow
- Beta (3-year or 5-year)
- Recent news headlines and sentiment summary (last 30 days)
- Analyst consensus rating and median price target
- **3-Month and 6-Month Relative Price Strength** — pull via bigdata.com to assess momentum posture. Compute the stock's price return over each window and compare it to the relevant benchmark (S&P 500 or TSX Composite, as applicable). Positive relative strength on both windows is a signal of sustained institutional accumulation and positive trend tailwinds. A stock that lags on both windows should receive a low Momentum Factor score unless the thesis explicitly calls for a mean-reversion setup.

### Technology Companies (additional)
- Cash and short-term investments on balance sheet
- R&D spend as % of revenue
- Gross margin trend (3-year)
- Segment revenue breakdown (cloud, hardware, software, services)
- Customer concentration risk (if disclosed)
- **Stock-Based Compensation (SBC) as % of Revenue** — tech firms routinely pay employees in stock, which does not appear in operating cash flow and therefore inflates reported FCF. Pull SBC from the cash flow statement and compute it as a % of revenue; flag any company above 10% as a meaningful dilution risk. Adjust FCF figures to SBC-adjusted FCF (FCF minus SBC) for apples-to-apples comparisons across names.
- Identify and describe the **secular growth driver** — what structural, multi-year trend is this company riding? (AI infrastructure, cloud migration, digital payments, etc.)

### Financial Companies — Banks and Insurers (additional)
- **Total Equity-to-Total Assets Ratio (Common Equity / Total Assets)** — use as a proxy for capital adequacy in place of CET1, which is not available from bigdata.com; flag any bank below 7% as potentially undercapitalized
- **Return on Equity (ROE) trend** (last 3–4 quarters) — use in place of NIM as the primary measure of earnings efficiency; declining ROE in a higher-for-longer environment signals asset repricing pressure or rising funding costs
- **Return on Assets (ROA) trend** (last 3–4 quarters) — use in place of PCL as a proxy for overall credit and operational health; ROA compression often reflects rising provisions or net interest spread compression before PCL is separately disclosed
- **Deposit stability** — note if recent quarters show deposit outflows or unusual deposit mix shift toward higher-cost funding
- Non-performing loans (NPL) ratio
- Non-performing loans (NPL) ratio
- For insurers: combined ratio, investment portfolio yield

> **Data note:** CET1 ratios and Net Interest Margin (NIM) / PCL are not available from bigdata.com tearsheets. The substitutions above (Equity/Assets, ROE trend, ROA trend) are the approved proxies for this skill — document any gap explicitly in the Data Confidence field of the output block.

### Industrial Companies (additional)
- **Asset Turnover Ratio (Revenue / Total Assets)** — use as a proxy for backlog/order-book health; rising asset turnover indicates the company is deploying its capital base productively and sustaining demand; declining asset turnover signals slowing orders or excess capacity
- **Inventory-to-Revenue Trend** (last 3–4 quarters) — a rising ratio may signal order softening or production ahead of demand; a stable or declining ratio alongside steady revenue is a positive indicator of operational discipline *(Backlog and order book are not available from bigdata.com for industrial names such as ETN; use these proxies and flag the gap in Data Confidence)*
- Operating leverage (sensitivity of operating income to revenue changes)
- Capital expenditure as % of revenue
- Exposure to government/infrastructure contracts vs. private sector
- Cyclicality assessment: does this company's revenue correlate strongly with GDP or industrial production?

### Communication Services (additional)
- Subscriber growth or churn rate (where applicable)
- ARPU (average revenue per user) trend
- Content/platform moat assessment
- Regulatory risk (spectrum, antitrust, content regulation)
- **Net Debt / EBITDA** — telecom and streaming companies (BCE, RCI.B, NFLX, DIS) routinely carry massive debt loads to fund 5G/fiber buildouts or content libraries. In a higher-for-longer rate environment this leverage is non-negotiable to monitor; flag any name above 3.5× as elevated risk and above 5.0× as a hard caution.
- **Content Spend Amortization Rate** *(streaming companies only — NFLX, DIS)* — the rate at which a company expenses its capitalized content production costs. A slow amortization schedule defers costs and artificially inflates current-period earnings. Pull total content amortization as a % of total content spend; flag companies where amortized content costs are growing materially slower than total content obligations.

---

## Step 4 — Sector-Level Evaluation Logic

Apply the following framework by sector. Do not deviate from it.

### Technology
A technology company **passes** if it demonstrates all three:
1. **Secular growth driver** — participates in a structural, multi-year growth trend (AI, cloud, enterprise software, semiconductors for AI workloads). Growth tied only to economic expansion does not qualify.
2. **Revenue growth** — TTM revenue growth of at least mid-single digits YoY; accelerating or stable growth preferred over decelerating.
3. **Balance sheet strength** — net cash position (cash > debt) or at minimum a manageable debt load with strong FCF coverage. Companies burning cash without a credible path to profitability are excluded.

*Additional flag:* In a higher-for-longer rate environment, high-multiple tech is rate-sensitive. For any tech name with a forward P/E above 35×, explicitly assess whether the growth runway justifies that premium and whether the company's cash position provides a buffer if sentiment rotates.

### Financials — Large Banks
A bank **passes** if it demonstrates all three:
1. **Total Equity-to-Total Assets Ratio ≥ 7%** — used as a capital adequacy proxy in place of CET1 (not available from bigdata.com); indicates the bank is not excessively leveraged relative to its asset base.
2. **Stable or improving ROE trend** — in a higher-for-longer environment, banks with strong deposit franchises and variable-rate loan books should show stable or expanding ROE; declining ROE is the observable proxy for NIM compression or funding cost pressure.
3. **Stable or improving ROA trend** — used as the primary proxy for credit quality in place of PCL; a declining ROA signals rising costs, provisions, or spread compression that may foreshadow earnings misses.

*Preference:* Large, diversified banks over monoline lenders. Canadian Big Six banks and U.S. money-center banks (JPM, BAC) are the primary candidates.

### Industrials
An industrial company **passes** if it demonstrates:
1. **High-quality, recurring or contracted revenue** — prefer companies with long-term contracts, government clients, or essential services (rail, waste management, engineering services).
2. **Low cyclicality or diversified exposure** — avoid pure-play industrial cyclicals with no pricing power.
3. **Manageable leverage** — Net Debt / EBITDA below 3.0×.
4. **Strong FCF conversion** — EBITDA-to-FCF conversion above 50%.

### Consumer Staples / Defensive Healthcare (if thesis permits)
These are considered only if the thesis explicitly calls for defensive ballast. Evaluate on: pricing power, dividend reliability, margin stability, and whether the current valuation is fair given the defensive premium.

---

## Step 5 — Scoring and Ranking

Score each company in the Eligible Universe that passes sector-level filters on the following dimensions. Each criterion is scored **0–100**, then multiplied by its weight to produce a weighted contribution. Sum all weighted contributions for a final score out of 100.

| Criterion | Weight | What a high score looks like |
|---|---|---|
| Thesis alignment (does this holding reinforce the stated macro view?) | 27% | Directly and specifically supports the thesis; low contradictions |
| Fundamental quality (balance sheet, margins, SBC-adjusted FCF) | 23% | Strong balance sheet, high margins, clean FCF, low leverage |
| Growth trajectory (revenue, earnings, secular driver strength) | 23% | Accelerating revenue, identifiable secular tailwind, earnings inflecting |
| Valuation reasonableness (flag extremes, not a price prediction) | 9% | Trading at a fair-to-modest premium to peers; not stretched vs. growth rate |
| Risk / beta (penalize elevated beta unless thesis is explicitly high-risk) | 8% | Low beta relative to peers; limited tail-risk scenarios |
| **Momentum Factor** (3-month and 6-month relative price strength vs. benchmark) | **10%** | Positive relative strength on both windows; consistent trend with no sign of institutional distribution |

> **Momentum Factor — Scoring Notes:** The Momentum Factor carries a strict maximum of 10% of the total score. The remaining 90% is anchored to the fundamental, valuation, and balance sheet criteria above. Evaluate 3-month and 6-month relative price strength pulled via bigdata.com. Both windows in positive relative outperformance vs. the benchmark = high score (80–100). Mixed signals (one window positive, one negative) = mid-range score (40–60). Both windows underperforming = low score (0–30). The fund's 30-day investment horizon means near-term trend confirmation is relevant but must not override fundamental quality — the Momentum Factor is a tie-breaker and tailwind signal, not a primary driver.
>
> **Risk Guardrail for Overextended Assets:** If a stock exhibits exceptional 6-month price performance (e.g., top quartile relative strength) **and** its TTM P/E trades more than 2 standard deviations above its own 3-year historical average P/E, the skill must flag this in the **Key Risk** column as: *"Potential Valuation Peak / Overcrowded Momentum Trade."* In this scenario, cap the Momentum Factor sub-score at 50 regardless of price strength to avoid rewarding a potentially exhausted trend. Document the TTM P/E, the estimated 3-year average P/E, and the deviation explicitly in the Key Metrics block.

Compute the final weighted score (0–100) for each candidate and sort descending. Aim to surface the **top 8–12 candidates** for the final output. Flag the bottom tercile explicitly as "monitor but do not recommend."

---

## Step 6 — Output Format

For each recommended company, output the following block:

---

### [Ticker] — [Company Name] | [Sector] | [Exchange]

**Thesis Alignment:** [One sentence: why this holding fits the macro thesis]

**Qualitative Rationale:** [2–3 sentence paragraph covering the company's competitive position, why now is the right entry context given the macro backdrop, and any relevant risk the investor should be aware of]

**Key Metrics:**
- Price: $X | 52-Wk Range: $X – $X
- Market Cap: $XB
- Revenue (TTM): $XB | YoY Growth: X%
- FCF (TTM): $XB | Net Debt/EBITDA: X.Xx
- P/E (TTM): Xx | YoY Revenue Growth: X% | Beta: X.X
- [Sector-specific metric 1]: X
- [Sector-specific metric 2]: X
- Analyst Consensus: [Buy / Hold / Sell] | Price Target: $X

**Data Confidence:** [High / Medium / Low — note if any metrics were missing or potentially stale from bigdata.com]

---

After all individual blocks, output a **Portfolio Summary Table** both inline in chat and written to Excel (see Excel Output instructions below):

| Ticker | Company | Sector | Thesis Role | Score (/100) | Suggested Weight Range | Key Risk | Where did the data or the AI's company picks look wrong or risky? |
|---|---|---|---|---|---|---|---|
| ... | ... | ... | Core / Tactical / Ballast | XX | X–X% | ... | ... |

**Thesis Role definitions:**
- **Core** — central to the thesis, high conviction, likely larger weight
- **Tactical** — fits the thesis but with a specific near-term catalyst or higher volatility; size accordingly
- **Ballast** — lower beta, provides stability; appropriate if mandate allows defensive positioning

---

### Excel Output — Portfolio Picks Compendium

After generating the Portfolio Summary Table, append it to `portfolio_picks_compendium.xlsx` in the same folder as this skill file using the following rules:

1. **If the file does not exist**, create it with a first sheet named `v1`.
2. **If the file already exists**, inspect existing sheet names, determine the next version number (e.g., if `v1` and `v2` exist, create `v3`), and add a new sheet with that name. Never overwrite an existing sheet.
3. Each sheet must contain the following columns in order: `Run`, `Ticker`, `Company`, `Sector`, `Thesis Role`, `Score (/100)`, `Suggested Weight Range`, `Key Risk`, `Where did the data or the AI's company picks look wrong or risky?`. The `Run` column should record the sheet name (e.g., `v1`) so that rows remain identifiable if sheets are ever copied or merged. The scrutiny column is populated in Step 7.5 after picks are finalized.
4. Apply light formatting: bold the header row, freeze the top row, and auto-fit column widths.
5. After writing the file, confirm in your response: *"Portfolio summary written to portfolio_picks_compendium.xlsx — [sheet name]."*

---

## Step 7 — Integrity Checks

Before finalizing output, run these checks:

1. **Universe compliance** — confirm every ticker in the output appears in `universe.csv`. Remove any that do not.
2. **Exclusion compliance** — confirm no hard-excluded company has been recommended.
3. **Position limit** — flag any recommendation where a reasonable single-name weight would exceed 25%.
4. **Sector diversification** — confirm recommendations span at least 3 sectors.
4b. **Sector concentration compliance** — verify that: (a) no single GICS sector exceeds 35% of portfolio weight; (b) Technology + Communication Services combined does not exceed 50%; (c) no investment sub-theme exceeds 30%; (d) no GICS sub-industry contains more than 2 picks. If any limit is breached, resolve before finalizing — drop or trim the offending positions and re-confirm weights sum to 100%.
5. **Data gaps** — list any metric that could not be retrieved from bigdata.com and note how it affected the analysis.
6. **Stale data warning** — if any bigdata.com data appears more than 5 trading days old, flag it explicitly rather than presenting it as current.

---

## Step 7.5 — Third-Party Scrutiny Pass

**Run this step after completing the full output in Step 6. Do not let it influence the picks or scores — produce those independently first, then apply this lens.**

Adopt the perspective of a skeptical senior analyst reviewing this output for the first time, with no attachment to the recommendations. For each company in the final output, answer the question:

> *"Where did the data or the AI's company picks look wrong or risky?"*

Evaluate each pick against the following scrutiny criteria:

| Scrutiny Dimension | What to look for |
|---|---|
| **Internal rule violations** | Did this pick fail a hard pass/fail criterion (e.g., leverage > 3.0×, SBC > 10%, unresolved capital adequacy check) without a documented override? Flag any case where a threshold was breached and the pick was included anyway. |
| **Score generosity** | Does the score feel inflated relative to the actual data? Identify any dimension (thesis alignment, quality, growth, valuation, risk) where a high sub-score was awarded despite thin or missing evidence. |
| **Data gap exploitation** | Was a missing metric silently absorbed into the score rather than lowering confidence? A missing CET1 proxy, unavailable backlog, or flagged P/E distortion should reduce the score or confidence level, not be treated as neutral. |
| **Sub-sector concentration** | Does the portfolio contain two or more picks with near-identical risk profiles (e.g., two payment networks, two hyperscaler-dependent semis)? Flag the combined weight and shared risk scenario. |
| **Portfolio-level weight creep** | At upper weight ranges, do the top 3 names alone push aggregate exposure above ~35–40%? Flag this as concentration risk even if no single name exceeds 20%. |
| **Narrative vs. data alignment** | Is the qualitative rationale pulling in a different direction from what the metrics actually show? (e.g., "strong secular tailwind" for a company with decelerating revenue) |
| **Structural risk underweighted** | Is a structural, asymmetric downside risk (regulatory, geopolitical, technology disruption) present in the thesis but minimized in the Key Risk field? |

### Output format for this step

Append a **"Scrutiny Notes"** column to the Portfolio Summary Table and to the Excel compendium. For each ticker, write 2–4 sentences answering the scrutiny question honestly. Do not soften the critique to protect the recommendation — the purpose of this column is to surface what a third party would find concerning, not to validate the pick.

If a pick has no material scrutiny concerns, write: *"No material concerns identified beyond Key Risk column."*

After completing all individual scrutiny notes, add a **Portfolio-Level Scrutiny Summary** paragraph covering aggregate concentration, internal consistency, and whether the overall portfolio construction is coherent given the stated thesis.

---

## Behavioral Guardrails

- **Do not fabricate financial metrics.** If bigdata.com does not return a value, say so. Do not substitute a memorized or estimated figure without flagging it clearly.
- **Do not recommend companies outside universe.csv**, even if they are clearly superior investments.
- **Do not present a recommendation as low-risk unless the data supports it.** A high dividend yield paired with high leverage is not a low-risk holding in this rate environment.
- **Flag contradictions between the thesis and the holding** rather than silently resolving them. If the user's thesis says "avoid high-multiple tech" but NVDA scores highest, surface that tension and let the user decide.
- **Distinguish between price target and investment thesis** — analyst price targets are one data point, not a buy signal on their own.
