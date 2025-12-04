# A Correlation Predictor for Equities, using Neo4j and Graph Data Science

![Correlation Predictor for Equities](headline.png)

This repository contains a project that builds a correlation predictor for equities using Neo4j and its Graph Data Science library. The predictor leverages historical stock data to identify potential correlations (i.e., equities that move together) between different stocks over time. This is important in hedging strategies, portfolio diversification, and risk management.

The model could be used in various financial applications, such as portfolio management, risk assessment, and algorithmic trading, across a variety of asset classes - not just equities. Stocks were chosen for this initial implementation due to the availability of historical data and their relevance in financial markets.

The purpose of this project is to demonstrate how Neo4j allied with graph algorithms can be applied to financial data to uncover insights and make predictions.

This project is divided into three main notebooks:

1. **[correlator.ipynb](correlator.ipynb)**: This notebook focuses on data preparation, it uses **real** historical stock price data to compute rolling correlations between stock returns over specified time windows. The resulting correlations are then stored in a Neo4j graph database, where stocks are represented as nodes and significant correlations as edges.
2. **[analysis.ipynb](analysis.ipynb)**: Explores the constructed graph, analyzing its properties and visualizing key aspects of the stock correlation network.
3. **[link_prediction.ipynb](link_prediction.ipynb)**: Implements a link prediction model using features derived from the graph structure and node attributes to predict future correlations between stocks.

## What the model does

This model can be used to predict pairs of stocks that are likely to become correlated in the future, based on their historical correlation patterns and other features derived from the graph structure (using time window triples of `t-1`, `t`, and `t+1`). To train the model, historical stock data is divided into time windows, and for each window, a correlation graph is constructed where nodes represent stocks and edges represent significant correlations between them. The model is trained to predict the formation of edges (correlations) in future time windows based on features extracted from the graph structure and node attributes:

- Node2Vec embeddings
- Degree (number of connections)
- Jaccard similarity (common neighbors)
- [PageRank](https://neo4j.com/docs/graph-data-science/current/algorithms/page-rank/) scores
- [Betweenness](https://neo4j.com/docs/graph-data-science/current/algorithms/betweenness-centrality/) centrality
- Stock-specific features (e.g., volatility, momentum)

The trained model can then be used to predict potential correlations between stocks in future time windows, aiding in investment decisions and risk management:

- **Which stocks are likely to move together in the future?**
- **How can we diversify a portfolio to minimize risk based on predicted correlations?**

With some adaptations, it could also answer:

- **Which stocks are emerging as "Super-Hubs" of risk?** By predicting which nodes will rapidly increase their degree centrality (connect to many others), you can identify stocks that, if they fail, could drag down a large portion of the market.

- **What are the likely pathways for sector contagion?** If the model predicts new links forming between a Tech stock and a Banking stock, it identifies a bridge where volatility in one sector could spill over into another.

- **Which uncorrelated pairs are ripe for Statistical Arbitrage?** The model identifies pairs that are not yet correlated but will likely become correlated. Traders can enter "pairs trades" early (long one, short the other) before the market consensus prices in the relationship.

- **Are we witnessing a "regime shift" in market behavior?** If the model suddenly predicts a massive increase in total graph density (everything correlating with everything), it signals a shift from a "stock-picker's market" to a macro-driven "risk-on/risk-off" environment.

- **Which assets are "false diversifiers"?** You might hold Stock A and Stock B thinking they are distinct. If the model predicts a high probability of a link forming next month, your portfolio is becoming less diversified than you think.

And many more...

## What the graph looks like

![Correlations for ABT](abt-corr.png)

The above graph shows correlations for the stock Abbott Laboratories (ABT) where the correlation threshold is `> 0.85`, as well as the time windows associated with each correlation, and the stocks ABT correlated with over time. It is equivalent to the following query:

```cypher
MATCH (abt:Stock {ticker:"ABT"})
MATCH (abt)-[cor:CO_MOVES_WITH]->(correlated:Stock)
WHERE cor.corr > 0.85

// Metrics for ABT in those same windows
OPTIONAL MATCH (abt)-[hm_abt:HAS_METRICS]->(abt_metrics:StockWindow {window_id: cor.window_id})

// Metrics for the correlated stock in those same windows
OPTIONAL MATCH (correlated)-[hm_corr:HAS_METRICS]->(corr_metrics:StockWindow {window_id: cor.window_id})

RETURN
  abt,
  correlated,
  cor,
  abt_metrics,
  hm_abt,
  corr_metrics,
  hm_corr
ORDER BY cor.window_id, cor.corr DESC;
```

Our model uses such graphs constructed over rolling time windows to learn patterns and predict future correlations. The time window sizes you ultimately choose will depend on your specific use case and the dynamics of the market you are analyzing, but the ability of the model to learn from historical patterns will vary depending on those choices - for example, shorter windows may suffer from noise, while longer windows may miss short-term dynamics. Different features may also be more or less relevant depending on the window size, feature engineering and selection will be key to a successful model.

## Model performance

Preliminary results indicate that the model can achieve significant predictive accuracy, especially considering the complexity and noise inherent in financial markets. Further tuning and feature engineering may enhance performance. For the run in this repository, an [XGBoost](https://xgboost.readthedocs.io/en/stable/) classifier was used, achieving the following metrics:

```
ROC AUC: 0.7377
Avg Precision: 0.0412
Baseline (Random Chance): 0.0118
Model AP: 0.0412
Lift: 3.48x (the model is 3.48x times better than random guessing in terms of average precision)
```

The above suggests the model is useable for practical applications, although further improvements could be made to make it more robust, and time invariant.
