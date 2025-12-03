# Notes on Stock Correlation Analysis

## Quick instructions on how to load the produced CSVs into Neo4j

1. Create a new Neo4j database (`stocks`).
2. Place the CSV files (`stocks.csv`, `co_movements_windows.csv`, `window_ticker_stats.csv`) in the `import` folder of the Neo4j database.
3. Open Neo4j Browser and run the Cypher commands below in order to create constraints and load the data.

## High-level overview

Node label: `Stock`
Relationship type: `CO_MOVES_WITH`
Key idea: edges are time-windowed correlations between returns.

### Handling time windows

- Simpler (good first pass): one graph per time window → store only the latest window in Neo4j.
- Richer: keep multiple windows and add window_id/start_date/end_date on edges.

## Node model

First, create uniqueness constraint on ticker:

```cypher
CREATE CONSTRAINT stock_ticker_unique IF NOT EXISTS
FOR (s:Stock)
REQUIRE s.ticker IS UNIQUE;
```

Load with:

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

## Relationship model

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
       toFloat(row.volatility_norm) AS volatilityNorm,
       toFloat(row.avg_volume)  AS avgVolume,
       toFloat(row.volume_norm) AS volumeNorm,
       toFloat(row.vol_zscore)  AS volZscore,
       toFloat(row.momentum)    AS momentum,
       toFloat(row.momentum_norm) AS momentumNorm

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
    w.volume_norm = volumeNorm,
    w.vol_zscore  = volZscore,
    w.momentum    = momentum,
    w.volatility_norm = volatilityNorm,
    w.momentum_norm   = momentumNorm

  MERGE (s)-[:HAS_METRICS]->(w)
} IN TRANSACTIONS OF 1000 ROWS;
```
