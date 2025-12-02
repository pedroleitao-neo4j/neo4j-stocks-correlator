# A Correlation Predictor for Equities, using Neo4j and Graph Data Science

This repository contains a project that builds a correlation predictor for equities using Neo4j and its Graph Data Science library. The predictor leverages historical stock data to identify potential correlations (i.e., equities that move together) between different stocks over time. This is important in hedging strategies, portfolio diversification, and risk management.

The model could be used in various financial applications, such as portfolio management, risk assessment, and algorithmic trading, across a variety of asset classes - not just equities. Stocks were chosen for this initial implementation due to the availability of historical data and their relevance in financial markets.

The purpose of this project is to demonstrate how Neo4j allied with graph algorithms can be applied to financial data to uncover insights and make predictions.

This project is divided into three main notebooks:

1. **[correlator.ipynb](correlator.ipynb)**: This notebook focuses on data preparation, it uses **real** historical stock price data to compute rolling correlations between stock returns over specified time windows. The resulting correlations are then stored in a Neo4j graph database, where stocks are represented as nodes and significant correlations as edges.
2. **[analysis.ipynb](analysis.ipynb)**: This notebook explores the constructed graph, analyzing its properties and visualizing key aspects of the stock correlation network.
3. **[link_prediction.ipynb](link_prediction.ipynb)**: This notebook implements a link prediction model using features derived from the graph structure and node attributes to predict future correlations between stocks.

## What the model does

This model can be used to predict pairs of stocks that are likely to become correlated in the future, based on their historical correlation patterns and other features derived from the graph structure. To train the model, historical stock data is divided into time windows, and for each window, a correlation graph is constructed where nodes represent stocks and edges represent significant correlations between them. The model is trained to predict the formation of edges (correlations) in future time windows based on features extracted from the graph structure and node attributes:

- Node2Vec embeddings
- Degree (number of connections)
- Jaccard similarity (common neighbors)
- Stock-specific features (e.g., volatility, momentum)

The trained model can then be used to predict potential correlations between stocks in future time windows, aiding in investment decisions and risk management.

Preliminary results indicate that the model can achieve significant predictive accuracy, especially considering the complexity and noise inherent in financial markets. Further tuning and feature engineering may enhance performance. For the run in this repository, an [XGBoost](https://xgboost.readthedocs.io/en/stable/) classifier was used, achieving the following metrics:

```
ROC/AP: (0.7554971431247659, 0.08409355145560418)
Baseline (Random Chance): 0.0118
Model AP: 0.0841
Lift from random classification: 7.10x (the model is 7.1 times better than random guessing in terms of average precision)
```

The above suggests the model is useable for practical applications, although further improvements could be made.
