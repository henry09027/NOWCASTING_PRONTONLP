---
name: Sentiment Anomaly & Narrative Analysis Skill
Description: This skill identifies statistically unusual sentiment shifts from earnings call transcripts across Russell 3000 and S&P 500 sectors, then builds equity-analyst-style narratives explaining what's driving those shifts. It combines Z-score anomaly detection with time-series trend visualization and external research to produce actionable investment insights.
---

## Available Tools

### 1. GET_SENTIMENT_MOVERS (Anomaly Detection)
**Purpose:** Identifies sector × event category combinations with statistically significant sentiment deviations from historical norms.

**Parameters:**
| Parameter | Type | Description | Example Values |
|-----------|------|-------------|----------------|
| `p_indexname` | VARCHAR | Financial index to analyze | `Russell 3000 Index`, `S&P 500 Index` |
| `p_documentsectionname` | VARCHAR | Section of earnings call document | `Total`, `Presentation`, `Question`, `Answer` |
| `p_importance` | VARCHAR | Importance level filter | `high`, `medium`, `low` |

**Returns:** A table ranked by MAX_ABS_ZSCORE (descending) with columns:
- `SECTOR` — Broad market sector
- `EVENTCATEGORYNAME` — Thematic category (e.g., Forecast_FinancialPerformance)
- `LATEST_QUARTER` — Most recent quarter in data
- `LATEST_QTR_AVG_SCORE` — Average net positivity for the latest quarter
- `TRAILING_4QTR_AVG_SCORE` — Rolling 4-quarter average
- `HISTORICAL_AVG_SCORE` — Long-run baseline mean
- `HISTORICAL_STDDEV` — Long-run standard deviation
- `LATEST_QTR_CHANGE` — Absolute change from historical mean
- `TRAILING_4QTR_CHANGE` — Trailing 4Q change from historical mean
- `LATEST_QTR_ZSCORE` — Z-score for latest quarter
- `TRAILING_4QTR_ZSCORE` — Z-score for trailing 4 quarters
- `MAX_ABS_ZSCORE` — Maximum of the two absolute Z-scores (used for ranking)

**Interpretation:**
- |Z-score| > 3.0 → Highly anomalous, warrants deep investigation
- |Z-score| > 2.0 → Notable deviation, monitor closely
- Positive Z → Sentiment is unusually bullish vs. history
- Negative Z → Sentiment is unusually bearish vs. history

---

### 2. GET_AVG_NET_POSITIVITY (Time Series)
**Purpose:** Retrieves quarterly time-series of average net positivity scores for a specific sector × event category × index combination.

**Parameters:**
| Parameter | Type | Description | Example Values |
|-----------|------|-------------|----------------|
| `p_indexname` | VARCHAR | Financial index | `Russell 3000 Index`, `S&P 500 Index` |
| `p_sector` | VARCHAR | Market sector | `Industrials`, `Financials`, `Information Technology`, `Communication Services`, `Consumer Discretionary`, `Health Care`, `Utilities`, `Materials`, `Energy`, `Real Estate`, `Consumer Staples` |
| `p_eventcategoryname` | VARCHAR | Event category label | See Event Categories section below |
| `p_documentsectionname` | VARCHAR | Document section | `Total`, `Presentation`, `Question`, `Answer` |
| `p_importance` | VARCHAR | Importance level | `high`, `medium`, `low` |

**Returns:** A table with columns:
- `CALENDARYEARQUARTER` — e.g., "2024-3"
- `AVG_NET_POSITIVITY_SCORE` — Score from -1 to 1

---

## Event Category Reference

Common `EVENTCATEGORYNAME` values:
- `CurrentState_FinancialPerformance`
- `Forecast_FinancialPerformance`
- `Surprise_FinancialPerformance`
- `CurrentState_OperationalPerformance`
- `Forecast_OperationalPerformance`
- `CurrentState_MarketAndCompetitivePosition`
- `Forecast_MarketAndCompetitivePosition`
- `StrategicPosition_RegulatoryAndLegalIssues`
- `Forecast_RegulatoryAndLegalIssues`
- `CurrentState_RegulatoryAndLegalIssues`
- `StrategicPosition_MacroeconomicFactors`
- `Forecast_MacroeconomicFactors`
- `CurrentState_ESG`
- `Forecast_ESG`
- `Surprise_ESG`
- `StrategicPosition_StrategicInitiatives`
- `Surprise_StrategicInitiatives`
- `CurrentState_StrategicInitiatives`
- `StrategicPosition_OperationalPerformance`
- `StrategicPosition_FinancialPerformance`
- `StrategicPosition_ESG`
- `Forecast_CapitalAllocation`
- `CurrentState_CapitalAllocation`
- `StrategicPosition_CapitalAllocation`
- `Surprise_MacroeconomicFactors`
- `Surprise_RegulatoryAndLegalIssues`
- `Surprise_MarketAndCompetitivePosition`
- `Surprise_OperationalPerformance`
- `CurrentState_MacroeconomicFactors`

---

## Workflow Steps

### Step 1: Identify Anomalies
Call `GET_SENTIMENT_MOVERS` with your desired index, document section, and importance level.

**Default starting point (recommended):**

p_indexname = 'Russell 3000 Index'
p_documentsectionname = 'Total'
p_importance = 'high'

**Filter results to focus on:**
- Top movers with |MAX_ABS_ZSCORE| > 2.5 (strong signals)
- Separate UPTICK (positive Z) from DOWNTICK (negative Z)
- Focus on 5-8 most interesting sector/theme pairs that tell a coherent macro story

### Step 2: Categorize the Signals
Group the top movers into narrative buckets:
- **Cyclical recovery signals** (e.g., Industrials + Financials financial performance surging)
- **Structural growth signals** (e.g., Communication Services competitive position)
- **Risk/stress signals** (e.g., Consumer Discretionary regulatory concerns)
- **Fatigue/reversal signals** (e.g., Information Technology surprise declining)

### Step 3: Research the "Why"
For each top mover, search the web for:
- `"{sector} sector {year} Q{quarter} earnings outlook {theme}"`
- `"{sector} {year} {relevant macro event: tariffs, rate cuts, AI spending, etc.}"`

Look for reputable sources: Fidelity, BlackRock, Goldman Sachs, Reuters, S&P Global, Deloitte, Morningstar, Schwab, FactSet.

### Step 4: Pull Time Series
For each interesting sector/theme combination, call `GET_AVG_NET_POSITIVITY` with matching parameters.

Use the same `p_indexname`, `p_documentsectionname`, and `p_importance` as Step 1, plus the specific `p_sector` and `p_eventcategoryname` from the anomaly.

### Step 5: Visualize
Create a **line chart with points** for each sector showing the quarterly time series. Use:
- Ordinal x-axis (CALENDARYEARQUARTER)
- Quantitative y-axis (AVG_NET_POSITIVITY_SCORE)
- Set y-axis domain appropriately for each sector's score range
- Title format: `"{Sector} — {EventCategory readable name} (Net Positivity)"`

### Step 6: Narrate Inflection Points
For each chart, identify 4-6 major inflection points and explain:
- **What happened** (the score movement)
- **When** (the quarter)
- **Why** (macro/sector event driving the shift)

Common inflection drivers to check:
- COVID-19 (2020-Q2)
- Post-COVID recovery (2021)
- Fed rate hike cycle (2022-Q2 to 2023)
- AI emergence / ChatGPT (2023-Q1)
- Rate stabilization (2024)
- Tariff regime / Policy shifts (2025-2026)
- AI capex boom/fatigue (2025-2026)

### Step 7: Summarize for Portfolio Action
Create a summary table with:
| Signal | Implication | Positioning |

---

## Output Format

Structure the final output as:

1. **Summary Table** — Top movers with Z-scores, direction indicators (🔺/🔻)
2. **Analyst Commentary** — One section per sector with:
   - What's driving this (1-2 sentences)
   - Why it makes sense (research-backed, 1 paragraph)
   - Time series chart
   - Inflection point analysis (bulleted timeline)
3. **Portfolio Takeaways** — Actionable table

---

## Tips & Best Practices

- **Start broad, then drill down:** Use `Total` document section first; if you need granularity, try `Presentation` (management's prepared remarks) vs. `Question` (analyst questions) vs. `Answer` (management's Q&A responses) to see where sentiment diverges.
- **High importance first:** Start with `high` importance to focus on material themes. Use `medium` for broader coverage.
- **Cross-validate upticks vs. downticks:** If one sector is surging in "Forecast_FinancialPerformance" but declining in "Surprise_StrategicInitiatives," that tells a nuanced story (strong results but fading innovation narrative).
- **Compare indices:** Run the same analysis for S&P 500 vs. Russell 3000 to see if small-cap sentiment diverges from large-cap.
- **Track quarter-over-quarter:** The TRAILING_4QTR_ZSCORE helps distinguish one-off spikes from sustained trends.
- **Z > 2 is notable; Z > 3 is headline-worthy; Z > 4 is rare and demands explanation.**

---

## Example Invocations

**Find all anomalies:**

GET_SENTIMENT_MOVERS(
    p_indexname = 'Russell 3000 Index',
    p_documentsectionname = 'Total',
    p_importance = 'high'
)

**Get time series for a specific finding:**

GET_AVG_NET_POSITIVITY(
    p_indexname = 'Russell 3000 Index',
    p_sector = 'Industrials',
    p_eventcategoryname = 'CurrentState_FinancialPerformance',
    p_documentsectionname = 'Total',
    p_importance = 'high'
)

**Analyst-focused Q&A section only:**

GET_SENTIMENT_MOVERS(
p_indexname = 'Russell 3000 Index',
p_documentsectionname = 'Answer',
p_importance = 'high'
)
