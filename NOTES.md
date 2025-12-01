# Notes on Stock Correlation Analysis

## High-level overview

Node label: `Stock`
Relationship type: `CO_MOVES_WITH`
Key idea: edges are time-windowed correlations between returns.

### Handling time windows

- Simpler (good first pass): one graph per time window → store only the latest window in Neo4j.
- Richer: keep multiple windows and add window_id/start_date/end_date on edges.

## Node model

```cypher
CREATE CONSTRAINT stock_ticker_unique IF NOT EXISTS
FOR (s:Stock)
REQUIRE s.ticker IS UNIQUE;
```

Loading with:

```cypher
LOAD CSV WITH HEADERS FROM 'file:///stocks.csv' AS row
WITH row
WHERE row.ticker IS NOT NULL AND row.ticker <> ''
MERGE (s:Stock {ticker: row.ticker})
ON CREATE SET
  s.industry = row.industry,
  s.sector   = row.sector,
  s.country  = row.country;
```

## Example properties

```cypher
Stock {
  ticker: STRING!,       // e.g. "AAPL"
  name: STRING,
  exchange: STRING,      // "NYSE", "NASDAQ", "LSE"
  sector: STRING,        // e.g. "Information Technology"
  industry: STRING,      // e.g. "Semiconductors"
  country: STRING,       // "US", "UK"
  marketCap: FLOAT,      // optional
  beta: FLOAT            // optional, for features
}
```

## Relationship model

```cypher
(:Stock)-[:CO_MOVES_WITH {
  corr:   FLOAT,     // Pearson correlation in window
  window: STRING,    // "2024-01-01_2024-03-31"
  start:  DATE,      // optional
  end:    DATE,      // optional
  sign:   STRING     // "POS" or "NEG"
}]-(:Stock)
```

Typically only keep edges where `ABS(corr) >= 0.7`. Also drop self-loops and near-duplicates.

Load with:

```cypher
LOAD CSV WITH HEADERS FROM 'file:///co_movements_windows.csv' AS row
MATCH (s1:Stock {ticker: row.src_ticker})
MATCH (s2:Stock {ticker: row.dst_ticker})
MERGE (s1)-[r:CO_MOVES_WITH {window_id: row.window_id}]->(s2)
ON CREATE SET
  r.corr  = toFloat(row.corr),
  r.start = date(row.start),
  r.end   = date(row.end),
  r.window = row.window,
  r.sign  = row.sign;
```

## Metrics model
```cypher
(:Stock {ticker})
  -[:HAS_METRICS]->
(:StockWindow {
   window_id,
   window,
   start,
   end,
   mean_return,
   volatility,
   avg_volume,
   vol_zscore,
   momentum
})
```
Load with:

```cypher
LOAD CSV WITH HEADERS FROM 'file:///window_ticker_stats.csv' AS row
CALL (row) {
  WITH row
  WITH row,
       date(row.start)   AS startDate,
       date(row.end)     AS endDate,
       toFloat(row.mean_return) AS meanReturn,
       toFloat(row.volatility)  AS volatility,
       toFloat(row.avg_volume)  AS avgVolume,
       toFloat(row.vol_zscore)  AS volZscore,
       toFloat(row.momentum)    AS momentum

  MERGE (s:Stock {ticker: row.ticker})
  MERGE (w:StockWindow {ticker: row.ticker, window_id: row.window_id})
  ON CREATE SET
    w.window = row.window,
    w.start  = startDate,
    w.end    = endDate
  SET
    w.mean_return = meanReturn,
    w.volatility  = volatility,
    w.avg_volume  = avgVolume,
    w.vol_zscore  = volZscore,
    w.momentum    = momentum

  MERGE (s)-[:HAS_METRICS]->(w)
} IN TRANSACTIONS OF 1000 ROWS;
```
