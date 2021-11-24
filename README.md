# gas_market_analysis
Various attempts to predict the price of natural gas

## Table of Contents
1. [Reserves](#Reserves)
2. [Time Series Prediction](#Time-Series-Prediction)
3. [Further Methods](#Further-Methods)

## Reserves

### Intro
The theses is that changes in natural gas reserves influence the market price.

### Data
- Reserves data from the [U.S. Energy Information Administration](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&ved=2ahUKEwip-7rl7Zz0AhUURuUKHUaYDmoQFnoECA4QAQ&url=https%3A%2F%2Fir.eia.gov%2Fngs%2Fngshistory.xls&usg=AOvVaw1K7aXs_TSzq-ovhYuVd8D4): file [ngshistory.xls](https://github.com/ambader/gas_market_analysis/blob/main/data/ngshistory.xls)

- Market data from [yahoo! finance](https://de.finance.yahoo.com/quote/NG%3DF/history?p=NG%3DF): file [N_2001-01-01_2021-11-11.csv](https://github.com/ambader/gas_market_analysis/blob/main/data/N_2001-01-01_2021-11-11.csv)

<details>
<summary>Data Prep Code</summary>

```python
import numpy as np
import pandas as pd

#read and slice data

ds = pd.read_csv("N_2001-01-01_2021-11-11.csv")
df = pd.read_excel("ngshistory.xls").iloc[6:]

ds = ds[["Date","Adj Close"]].rename(columns={"Date" : "ds","Adj Close" : "y"})
ds.ds = pd.to_datetime(ds.ds)
ds["res"] = np.ones(len(ds))

df = df.rename(columns={df.columns[0]:"ds",df.columns[-1]:"y"})
df = df[["ds","y"]]
df["ds"] = pd.to_datetime(df.ds.values)
```
```python
  #weekly data to daily

for i in df.ds[1:].index:
    zw = ds[ds.ds.between(df.ds[i-1],df.ds[i])]
    ds.iloc[zw.index,-1] = df.loc[i].y
```
```python
  #normalize data

tt = ds[ds.ds.between(pd.to_datetime("2012"),pd.to_datetime("2020"))].reset_index(drop=True)
tt.y = tt.y/max(tt.y)
tt.res = tt.res/max(tt.res)
tt = tt.set_index("ds")
 ```
</details>

### Plot
<details>
<summary>PLT Code</summary>

```python
import seaborn as sns
import matplotlib.pyplot as plt
sns.set_theme(style="darkgrid")
plt.rcParams['font.sans-serif'] = 'Liberation Mono'

fig = plt.figure(figsize=(21,9))
ax = fig.add_subplot()
sns.lineplot(data=tt.rename(columns={"y":"Gas_price","res":"US_reserves"}),dashes=False,palette=["#000099","#ff9900"])
plt.ylabel("norm.Values")
plt.title("Gas_price-US_reserves Correlation", size=55,color='#3b3b3b',pad=25)
fig.text(0.5, -0.05, "Pearson correlation "+str(np.round(np.corrcoef(tt.y,tt.res)[1,0],4)), fontsize=25, ha='center',color='#3b3b3b')
plt.tight_layout()
plt.savefig("price_res_corr.png",bbox_inches='tight',dpi=250)
plt.show()
```
</details>

![](https://github.com/ambader/gas_market_analysis/blob/main/img/price_res_corr.png)

### Conclusion
The extention and shrinkage of reserves follows a stable and seasonal cycle as the demand for gas rises in colder month. Yet, the correlation is small. The conlusion is:
- the market usually **anticipates** the supply or demand side effect of reserves
- therefore, reserves can only influence prices if its change **deviates from expectations**

## Time Series Prediction

### Intro
The theses is that market prices represent contain aggregated expectations and therefore contain all informations to predict itself. Thus, i'll use the [fbprophet](https://facebook.github.io/prophet/) to project the market price time series into future.

### Data
- again the market data from [yahoo! finance](https://de.finance.yahoo.com/quote/NG%3DF/history?p=NG%3DF): file [N_2001-01-01_2021-11-11.csv](https://github.com/ambader/gas_market_analysis/blob/main/data/N_2001-01-01_2021-11-11.csv)

### Plot
<details>
<summary>PLT Code</summary>

```python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
from fbprophet import Prophet
 ```
 ```python
 p_pred = pd.read_csv("N_2001-01-01_2021-11-11.csv")
p_pred = p_pred.rename(columns={"Date" : "ds","Adj Close" : "y"})
p_pred = p_pred[["ds","y"]]
p_pred.ds = pd.to_datetime(p_pred.ds)

m = Prophet(yearly_seasonality=True,daily_seasonality=False)
m.fit(p_pred[-1000:])
future = m.make_future_dataframe(periods=100)
future = future[[s.weekday()<5 for s in pd.to_datetime(future.ds)]].reset_index(drop=True)
forecast = m.predict(future)

forecast.ds = pd.to_datetime(forecast.ds)
forecast = forecast.set_index("ds")
p_pred = p_pred.set_index("ds")
 ```
 ```python
sns.set_theme(style="darkgrid")
plt.rcParams['font.sans-serif'] = 'Liberation Mono'

fig, ax = plt.subplots(figsize=(21,9))
ax.plot(p_pred[p_pred.index.isin(forecast.index)].y, marker='o', markersize=2,color="#000099",linewidth=4)
ax.plot(forecast.yhat, color="#ff9900")
ax.fill_between(forecast.index,forecast.yhat_upper,forecast.yhat_lower, alpha=0.2, color="#ff9900",linewidth=4)
plt.ylabel("$/MMBtu")
plt.title("Gas_price-Time_series-Prediction", size=40,color='#3b3b3b',pad=25)
plt.legend( loc='lower left', labels=['Market Observation', 'Prediction','Trend Uncertainty/Noise'])
plt.savefig("ts_pred_1.png",bbox_inches='tight',dpi=250)
 ```
 </details>

![](https://github.com/ambader/gas_market_analysis/blob/main/img/ts_pred_1.png?raw=true)

 ### Cross Validation
  
 <details>
<summary>compute cross_val</summary>

```python
from fbprophet.diagnostics import cross_validation
df_cv = cross_validation(m, initial='366 days', period='365 days', horizon = '180 days')
```
 
```python
from fbprophet.diagnostics import performance_metrics
df_p = performance_metrics(df_cv)
```
   
 
```python
co_tr = []

df_cv = df_cv.set_index("ds")

for i in p_pred.index:
    if i in df_cv.index:
        co_tr.append(8)
    else:
        co_tr.append(np.nan)
   
tt = p_pred
tt["co"]=co_tr
   
tt["yhat_lower"] = df_cv.yhat_lower
tt["yhat_upper"] = df_cv.yhat_upper
tt["yhat"] = df_cv.yhat
   
tt = tt[tt.index.isin(pd.date_range(pd.to_datetime("2012"),pd.to_datetime("2021")))]
   
df_p.index = np.arange(162)+19
cross_val_par = "mean squared error","root mean squared error","mean absolute error","mean absolute percentage error","median absolute percentage error","coverage of the upper and lower intervals"
```
   
</details>
  
 <details>
<summary>plot cross_val</summary>

```python
import seaborn as sns
import matplotlib.pyplot as plt
sns.set_theme(style="darkgrid")
plt.rcParams['font.sans-serif'] = 'Liberation Mono'

fig, ax = plt.subplots(figsize=(21,9),nrows=2, ncols=1)
ax[0].plot(tt.y, marker='o', markersize=2,color="#000099")
ax[0].plot(tt.yhat, color="#ff9900")
ax[0].bar(tt.index,tt.co, color="crimson", alpha=0.1,width=4.1)
ax[0].fill_between(tt.index,tt.yhat_upper,tt.yhat_lower, alpha=0.4, color="#ff9900")
ax[1].plot(df_p.index,df_p[df_p.columns[1:]])
ax[0].set_ylabel("$/MMBtu")
ax[1].set_xlabel("forwarded Days")
ax[0].set_title("Gas_price-Time_series-Cross_Validation", size=40,color='#3b3b3b',pad=25)
ax[0].legend( loc='upper right', labels=['Market Observation', 'Prediction','Trend Uncertainty/Noise'])
ax[1].legend( loc='upper left',labels=cross_val_par)
```
</details>
  
 ![](https://github.com/ambader/gas_market_analysis/blob/main/img/price_cross_val.png?raw=true)
 
### Multiple Regressors
  
#### Intro
  
Using market data of gas business companies to refine the gas price prediction. The idea is that their action influence the gas market, e.g. if revenues of exploration providers shrink, it is likely that supply will decrease in the long-term. This procedure has one argumentative weakness: *why should the market generate different expectations* ? And continuative, if different data contains different expectations, one should rather try to identify the better one than combining them.
Besides the immanent friction of markets, I have two arguments which may contradict this objection:

- stock markets are stronger psychological driven than commodity markets, those are rather influenced due to inelastic physical trade.
  
- thus stock market data incorporates a larger time horizont and its capital allocation determines the upcoming market situation.
  
#### Data
  
- again the market data from [yahoo! finance](https://de.finance.yahoo.com/quote/NG%3DF/history?p=NG%3DF): file [N_2001-01-01_2021-11-11.csv](https://github.com/ambader/gas_market_analysis/blob/main/data/N_2001-01-01_2021-11-11.csv)
  
- One large gas producer - [Gazprom](https://en.wikipedia.org/wiki/Gazprom) (market cap ~115bn$): file [O_2000-01-01_2021-11-14.csv](https://raw.githubusercontent.com/ambader/gas_market_analysis/main/data/O_2000-01-01_2021-11-14.csv)
  
- One large service provider - [Schlumberger](https://en.wikipedia.org/wiki/Schlumberger) (market cap ~40bn$): file [SLB_2001-11-15_2021-11-14.csv](https://raw.githubusercontent.com/ambader/gas_market_analysis/main/data/SLB_2001-11-15_2021-11-14.csv)
  
### Analysis
Looks good ? It may does...
  
# ...Coming Soon...
  
### Conclusion
Markets trade expectations, so market prices represent the aggregated expectations. These are a sufficient indicator for cyclical effects as boom-bust in exploration. But (sudden) supply- and demand-side incidence actuate both shifts in expectations and prices. Therefore, it is necessary to find meta indices (see [Further Methods](#Further-Methods)).

## Further Methods

- Use [shipping rates / spot freight markets](https://www.balticexchange.com/en/data-services/market-information0/tankers-services.html) to indicate (short-term) local supply

- (Satellite) observing [gas flares](https://www.ggfrdata.org/) to indicate (short-term) production/ rig activity

- [Automatic Identification System (AIS)](https://www.marinetraffic.com) to track tanker movement

- [Global Conflict Risk Index (GCRI)](https://op.europa.eu/en/publication-detail/-/publication/1c121597-07cc-11e8-b8f5-01aa75ed71a1/language-en) for producer countries to indicate (medium-term) production
