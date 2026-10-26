# **Statistical Arbitrage \- Pairs Trading Strategy on Indian Equities**

## **Overview**

This project implements a statistical arbitrage pairs trading strategy on Indian equities listed on the NSE, particularly 10 popular FMCG stocks. The core idea is to identify two stocks that are economically linked and historically move together (cointegrated), and profit from temporary divergences in their prices by betting that the gap will close.

The project covers the full quantitative pipeline: universe screening, statistical pair selection, signal generation, realistic backtesting with transaction costs, and rigorous out-of-sample validation to prevent overfitting.

---

**Methodology**

### **1\. Data Collection and Train/Test Split**

Daily closing prices were downloaded from Yahoo Finance using `yfinance` for a universe of Indian FMCG  stocks listed on the NSE (`.NS` suffix). Data spans **2021-06-01 to 2026-06-01**.

The dataset was split before any analysis:

* Training: 2021-06-01 to 2024-06-01 — all pair selection, parameter tuning, and strategy development  
* Test: 2024-06-01 to 2026-06-01 — touched exactly once, after the strategy was fully finalised for the stock pair that provided the best results in training. 

This strict separation prevents data snooping bias

**2\. Stationarity Filter (ADF Test)**

Each individual price series was tested for stationarity using the Augmented Dickey-Fuller (ADF) test. Stock prices are typically non-stationary (random walks), so this step filters out any tickers whose prices happen to be stationary on their own, a prerequisite for meaningful cointegration testing. 

### **3\. Pair Selection via Cointegration**

Every possible pair from the surviving universe was tested for cointegration using the Engle-Granger test (`statsmodels.tsa.stattools.coint`). Two stocks are cointegrated if a linear combination of their prices is stationary,  meaning their spread reverts to a stable mean over time rather than drifting permanently.

Pairs were ranked by p-value. Only pairs with **p \< 0.05** were considered viable for trading.

### **4\. Hedge Ratio Estimation (OLS Regression)**

For each cointegrated pair, a hedge ratio was estimated by regressing one stock's price on the other using Ordinary Least Squares (OLS). This determines how many units of stock A to hold per unit of stock B so the combined position is truly spread-neutral. Accounting for the fact that the two stocks don't move dollar-for-dollar.

**Spread \= PriceA \- β × PriceB**

Where β (the slope coefficient) is the hedge ratio.

### **5\. Rolling Z-Score Signal Generation**

The raw spread was normalised into a rolling z-score using a 30 day lookback window to prevent lookahead bias:

Z \= (Spread \- Rolling Mean) / Rolling Standard Deviation

Trading signals were generated as follows:

* **Z \> \+2.0** → Short the spread (Stock A expensive relative to B)  
* **Z \< \-2.0** → Long the spread (Stock A cheap relative to B)  
* **|Z| \< 0.5** → Exit position (spread normalised)  
* **|Z| \> 4.0** → Stop loss (spread moved further against position)

The stop-loss threshold was tuned by testing values of \[2.5, 3.0, 3.5, 4.0, 4.5\] on training data only, selecting stop=4.0 based on optimal risk-adjusted performance. The same value was applied unchanged to the test period.

### **6\. Backtesting Engine**

A vectorised backtesting engine simulates the strategy day by day on historical data. Key design decisions:

**Signal shift by 1 day:** Signals observed at day T's close are executed at day T+1's open. This prevents the strategy from trading on prices it could not have acted on in reality.

**P\&L calculation:**

PNL \= Signal × Capital × ReturnsA − Signal × Capital × Hedge\_Ratio × ReturnsB

When signal \= \+1 (long spread): long Stock A, short Stock B. When signal \= \-1 (short spread): short Stock A, long Stock B. The sign of the signal automatically handles the long/short direction without requiring explicit conditional logic.

**Transaction costs and slippage:**

* Commission: 0.1% per leg  
* Slippage: 0.05% per leg  
* Total friction: 0.15% per leg, applied only on days where the position changes

### **7\. Performance Metrics**

* **Sharpe Ratio** — annualised return per unit of risk (×√252 for daily data)  
* **Maximum Drawdown** — largest peak-to-trough decline in the equity curve  
* **Total Return** — cumulative strategy return over the period  
* **Win Rate** — percentage of active trading days with positive P\&L  
* **Total Trades** — number of position changes (entries \+ exits)

---

## **Results**

| Metric  | Training (2021-24) | Test ( 2024-26) |
| :---- | :---- | :---- |
| Sharpe ratio  | 1.00 | 0.45 |
| Max Drawdown | \-12.71% | \-25.09% |
| Total return  | 56.47% | 14.82% |
| Total trades  | 49 | 37 |
| Win rate  | 46.84% | 46.61% |

### The best-performing pair was **MARICO.NS/EMAMILTD.NS(p=0.0345)**, selected based on the highest training Sharpe among all cointegrated pairs 

### **Interpretation**

The strategy achieved a training Sharpe of 1.00, which is the conventional threshold.  A maximum drawdown of \-12.71% on training indicates the strategy recovers losses relatively quickly, with limited peak-to-trough decline over the period. 

Out-of-sample results show a Sharpe of 0.45 with a positive total return of 14.82% on completely unseen data. The near identical win rates across training (46.84%) and test (46.61%) are the most encouraging signal. This consistency suggests the mean reversion edge is structurally persistent rather than period specific noise.

The increase in maximum drawdown is the primary risk concern for live deployment.

---

## **Key Findings**

**What worked:**

* Indian equities in stable sectors (FMCG/Banking) produced genuine cointegration, unlike the semiconductor universe, which was also tested and yielded only 2 viable pairs out of 91 due to structural volatility in 2021-2024  
* The rolling z-score signal captured real mean-reversion behaviour, as confirmed by out-of-sample performance  
* Transaction costs reduced gross returns but did not eliminate the strategy's edge, suggesting a robust underlying signal

---

## **Limitations and Future Work**

| Limitation  | Description  | Proposed solution |
| :---- | :---- | :---- |
| Static hedge ratio  | OLS hedge  ratio estimated once on training data, held fixed throughout | Kalman filter for dynamic hedge ratio estimation |
| Fixed position sizing  | All trades use identical notional capital | Volatility scaled position sizing. |
| Single pair  | Strategy runs on one pair at a time.  | Multi-pair portfolio with correlation-adjusted allocations.  |
| Daily frequency  | Daily closing prices miss intraday opportunities | Tick or hourly data for higher-frequency implementation |

**Tech Stack**

| Tool  | Purpose  |
| :---- | :---- |
| Yfinance  | Historical price data download  |
| Pandas  | Data manipulation and time series handling  |
| Numpy  | Vectorised numerical computation  |
| Statsmodel  | OLS regression, ADF test, cointegration  |
| Matplotlib  | Visualisation  |

---

## **Setup and Usage**

pip install yfinance pandas numpy statsmodels matplotlib

Open `statistical_arbitrage.ipynb` and run all cells in order. The notebook is structured as a linear pipeline. 

---

## **Project Structure**

statistical-arbitrage-pairs-trading/  
│  
├── statistical\_arbitrage.ipynb   \# main notebook — full pipeline  
└── README.md                     \# this file

---

## **References**

* Chan, E. (2009). *Quantitative Trading*. Wiley.  
* Chan, E. (2013). *Algorithmic Trading: Winning Strategies and Their Rationale*. Wiley.  
* Engle, R. F., & Granger, C. W. J. (1987). Co-integration and error correction. *Econometrica*, 55(2), 251–276.

