---
Skill Name: sentiment-movers-analysis
Description; This skill identifies statistically anomalous sentiment shifts across sector × event category combinations in financial earnings calls, provides equity analyst-style interpretation backed by external research, generates time-series visualizations of sentiment trends, and extracts specific earnings call excerpts as evidence. It leverages two Snowflake table-valued functions (GET_SENTIMENT_MOVERS, GET_AVG_NET_POSITIVITY) and Pronto MCP search tools.
---

## When to Use
- User asks about **unusual sentiment shifts**, **sentiment movers**, or **anomalies** in earnings call data
- User wants to understand **why** a sector's sentiment on a specific theme changed significantly
- User requests a combined quantitative + qualitative analysis of sector-level sentiment patterns
- User asks for **nowcasting signals** or **earnings sentiment monitoring**

## Prerequisites
- Access to `QRSLLM_POC_DB.NOWCASTING_SANDBOX` schema
- SELECT privileges on `PRONTO_OVERALL_ATCLEVELNETPOSITIVITY_INFO`
- USAGE rights on `GET_SENTIMENT_MOVERS` and `GET_AVG_NET_POSITIVITY` functions
- Access to Pronto MCP search tools for earnings call excerpts

---

## Workflow Steps

### Step 1: Identify Anomalous Sentiment Movers

**Tool:** `GET_SENTIMENT_MOVERS`

**Purpose:** Surface sector × event category combinations with statistically significant Z-score deviations from their historical norms.

**Parameters:**

| Parameter | Description | Valid Values |
|-----------|-------------|--------------|
| `P_INDEXNAME` | Financial index to filter | `Russell 3000 Index`, `S&P 500 Index` |
| `P_DOCUMENTSECTIONNAME` | Document section | `Total`, `Presentation`, `Question`, `Answer` |
| `P_IMPORTANCE` | Importance level | `high`, `medium`, `low` |

**Example Call:**
```
GET_SENTIMENT_MOVERS(
  P_INDEXNAME => 'Russell 3000 Index',
  P_DOCUMENTSECTIONNAME => 'Total',
  P_IMPORTANCE => 'high'
)
```

**Output Columns:**
- `SECTOR` — Market sector
- `EVENTCATEGORYNAME` — Event theme (e.g., `CurrentState_FinancialPerformance`, `Forecast_MarketAndCompetitivePosition`)
- `LATEST_QUARTER` — Most recent quarter (e.g., `2026-2`)
- `LATEST_QTR_AVG_SCORE` — Average net positivity in latest quarter
- `TRAILING_4QTR_AVG_SCORE` — Trailing 4-quarter average
- `HISTORICAL_AVG_SCORE` — Long-run historical mean
- `HISTORICAL_STDDEV` — Historical standard deviation
- `LATEST_QTR_CHANGE` — Absolute change from historical mean
- `LATEST_QTR_ZSCORE` — Z-score for latest quarter vs. history
- `TRAILING_4QTR_ZSCORE` — Z-score for trailing 4Q vs. history
- `MAX_ABS_ZSCORE` — Maximum absolute Z-score (used for ranking)

**Interpretation Rules:**
- **|Z| > 3.0** → Highly anomalous, almost certainly a material shift — MUST investigate
- **|Z| > 2.0** → Statistically significant — should investigate
- **|Z| > 1.5** → Notable but may be noise
- **Positive Z-score** → Sentiment UPTICK (more positive than historical norm)
- **Negative Z-score** → Sentiment DOWNTICK (more negative than historical norm)

**Selection Criteria for Deep Dive:**
Select the top 3–5 sector × event combinations based on:
1. Highest `MAX_ABS_ZSCORE` (statistical significance)
2. Diversity of sectors (don't pick 5 from the same sector)
3. Mix of upticks AND downticks for contrasting narratives
4. Relevance to current market themes

---

### Step 2: Equity Analyst Interpretation + Web Research

**Tool:** `Web_Search`

**Purpose:** Build a macro narrative explaining WHY each selected sector is showing anomalous sentiment. Act as an equity analyst providing context.

**Search Strategy:**
For each selected sector × event combination, search for:
- `"[sector] sector Q[quarter] [year] earnings [event theme keywords]"`
- `"[sector] [year] [relevant macro driver: tariffs/AI/reshoring/etc.]"`

**Analyst Framework:**
For each anomalous signal, provide:
1. **Thesis** — One-sentence summary of the narrative
2. **Key Catalysts** — 2-3 specific drivers (with data points from web sources)
3. **Supporting Evidence** — Analyst reports, macro data (PMI, spending figures, etc.)
4. **Implications** — What this means for sector positioning

---

### Step 3: Generate Time-Series Sentiment Data

**Tool:** `GET_AVG_NET_POSITIVITY`

**Purpose:** Retrieve the full quarterly time-series for each selected sector × event combination to visualize trends and identify inflection points.

**Parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `P_INDEXNAME` | Index name | `Russell 3000 Index` |
| `P_SECTOR` | Sector name (exact match) | `Industrials`, `Information Technology` |
| `P_EVENTCATEGORYNAME` | Event category (exact match) | `CurrentState_FinancialPerformance` |
| `P_DOCUMENTSECTIONNAME` | Section name | `Total` |
| `P_IMPORTANCE` | Importance level | `high` |

**Example Call:**
```
GET_AVG_NET_POSITIVITY(
  P_INDEXNAME => 'Russell 3000 Index',
  P_SECTOR => 'Industrials',
  P_EVENTCATEGORYNAME => 'CurrentState_FinancialPerformance',
  P_DOCUMENTSECTIONNAME => 'Total',
  P_IMPORTANCE => 'high'
)
```

**Output Columns:**
- `CALENDARYEARQUARTER` — Quarter label (e.g., `2020-1`, `2026-2`)
- `AVG_NET_POSITIVITY_SCORE` — Average net positivity score for that quarter

**IMPORTANT:** Call this function ONCE per selected sector × event combination. Make all calls in parallel (no dependencies between them).

---

### Step 4: Create Visualizations

**Tool:** `data_to_chart` (Vega-Lite)

**Purpose:** Create one line chart per sector showing the time-series evolution of sentiment.

**Chart Specification Template:**
```json
{
  "title": "[Sector]: [Event Category Human-Readable Name] (R3K, High Importance)",
  "mark": {"type": "line", "point": true, "strokeWidth": 2},
  "encoding": {
    "x": {
      "field": "CALENDARYEARQUARTER",
      "type": "ordinal",
      "title": "Quarter",
      "axis": {"labelAngle": -45}
    },
    "y": {
      "field": "AVG_NET_POSITIVITY_SCORE",
      "type": "quantitative",
      "title": "Avg Net Positivity Score",
      "scale": {"zero": true}
    }
  }
}
```

**Inflection Point Analysis:**
After generating each chart, identify and explain:
- Major peaks and troughs
- Trend reversals (inflection points)
- The current trajectory relative to history
- Link each inflection to a known macro event (COVID, rate hikes, AI boom, tariffs, etc.)

---

### Step 5: Extract Earnings Call Evidence

**Tool:** `pronto_mcp_search`

**Purpose:** Find specific management quotes from earnings calls that explain the sentiment shift.

**Search Strategy per Sector:**

| For Uptick Sectors | For Downtick Sectors |
|-------------------|---------------------|
| `sentiment: "positive"` | `sentiment: "negative"` |
| Search for growth/strength keywords | Search for risk/uncertainty keywords |

**Parameters Template:**
```json
{
  "sections": ["EarningsCalls_PresenterSpeech"],
  "sectors": ["[target sector]"],
  "sentiment": "[positive or negative based on Z-score direction]",
  "sinceDay": "[start of the anomalous quarter, e.g., 2026-01-01]",
  "untilDay": "[today's date]",
  "size": 10,
  "topicSearchQuery": "[natural language description of the theme]"
}
```

**Topic Search Query Examples by Event Category:**
| Event Category | Positive Search Query | Negative Search Query |
|---------------|----------------------|----------------------|
| `CurrentState_FinancialPerformance` | `record revenue strong financial results growth accelerating` | `revenue decline earnings miss weak demand` |
| `Forecast_MarketAndCompetitivePosition` | `market leadership competitive advantage growing market share` | `competitive pressure market share loss disruption` |
| `StrategicPosition_RegulatoryAndLegalIssues` | `regulatory clarity favorable policy compliance` | `tariffs regulatory uncertainty legal challenges trade policy` |
| `Forecast_FinancialPerformance` | `strong outlook raised guidance revenue growth acceleration` | `lowered guidance cautious outlook headwinds` |
| `CurrentState_OperationalPerformance` | `operational efficiency productivity improvement margin expansion` | `supply chain disruption operational challenges cost pressure` |

**Output Format:**
Present 2-3 direct quotes per sector with:
- Speaker role (CEO, CFO)
- Company name
- Date
- The actual quote text

---

## Valid Parameter Values Reference

### Index Names
- `Russell 3000 Index`
- `S&P 500 Index`

### Sectors (exact match required)
```
Communication Services
Consumer Discretionary
Consumer Staples
Energy
Financials
Health Care
Industrials
Information Technology
Materials
Real Estate
Utilities
```

### Event Category Names (common ones)
```
CurrentState_FinancialPerformance
CurrentState_OperationalPerformance
CurrentState_MarketAndCompetitivePosition
CurrentState_StrategicInitiatives
CurrentState_RegulatoryAndLegalIssues
CurrentState_ESG
CurrentState_MacroeconomicFactors
CurrentState_CapitalAllocation
Forecast_FinancialPerformance
Forecast_OperationalPerformance
Forecast_MarketAndCompetitivePosition
Forecast_RegulatoryAndLegalIssues
Forecast_MacroeconomicFactors
Forecast_CapitalAllocation
Forecast_ESG
StrategicPosition_OperationalPerformance
StrategicPosition_RegulatoryAndLegalIssues
StrategicPosition_MacroeconomicFactors
StrategicPosition_StrategicInitiatives
StrategicPosition_CapitalAllocation
StrategicPosition_ESG
Surprise_FinancialPerformance
Surprise_OperationalPerformance
Surprise_MarketAndCompetitivePosition
Surprise_StrategicInitiatives
Surprise_RegulatoryAndLegalIssues
Surprise_ESG
Surprise_MacroeconomicFactors
```

### Document Section Names
- `Total` — All sections combined
- `Presentation` — Management presentation only
- `Question` — Analyst questions only
- `Answer` — Management answers to questions only

### Importance Levels
- `high`
- `medium`
- `low`

---

## Output Structure

The final output should follow this structure:

1. **Summary Table** — Top 5-10 anomalous signals ranked by |Z-score|
2. **Analyst Interpretation** — For each selected signal:
   - Thesis statement
   - Key catalysts (2-3 bullets with data)
   - Market implications
3. **Time-Series Charts** — One line chart per selected sector (4-5 charts typical)
   - Include inflection point annotations in the written analysis
4. **Earnings Call Evidence** — 2-3 direct quotes per sector from management
5. **Summary/Positioning Table** — Final table with signal direction and key driver per sector

---

## Error Handling

- If `GET_SENTIMENT_MOVERS` returns no results → Check parameter spelling (exact match required)
- If `GET_AVG_NET_POSITIVITY` returns empty → Verify sector name and event category are exact matches from Step 1 results
- If Pronto MCP search returns few results → Broaden the date range or simplify the topicSearchQuery
- All five filter parameters are required and applied as exact-match conditions — NULL or incorrect strings yield empty result sets

---

## Example Invocation Prompt

> "Use the GET_SENTIMENT_MOVERS function to find unusual sentiment shifts in the Russell 3000 Index, Total section, high importance. Then analyze the top movers as an equity analyst, generate time-series charts using GET_AVG_NET_POSITIVITY, and extract supporting earnings call quotes."

---

## Notes

- The functions require at least 4 historical quarters of data to compute valid Z-scores
- "Noisy" categories (Other, Fluff, generic labels) are automatically excluded by GET_SENTIMENT_MOVERS
- Results are ranked by `MAX_ABS_ZSCORE` descending — most anomalous at top
- Net positivity scores range from -1 to +1 (negative = more negative sentiment, positive = more positive)
- Quarter format is `YYYY-Q` (e.g., `2026-2` = Q2 2026)
