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
