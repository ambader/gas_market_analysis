# gas_market_analysis
Various attempts to predict the price of natural gas

## Table of Contents
1. [Reserves](#Reserves)
2. [Time Series Prediction](#Time-Series-Prediction)
3. [Further Ideas](#Further-Ideas)

## Reserves

### Intro
The theses is that changes in natural gas reserves influence the market price.

### Data
-Reserves data from the [U.S. Energy Information Administration](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&ved=2ahUKEwip-7rl7Zz0AhUURuUKHUaYDmoQFnoECA4QAQ&url=https%3A%2F%2Fir.eia.gov%2Fngs%2Fngshistory.xls&usg=AOvVaw1K7aXs_TSzq-ovhYuVd8D4): file [ngshistory.xls](https://github.com/ambader/gas_market_analysis/blob/main/data/ngshistory.xls)

-Market data from [yahoo! finance](https://de.finance.yahoo.com/quote/NG%3DF/history?p=NG%3DF): file [N_2001-01-01_2021-11-11.csv](https://github.com/ambader/gas_market_analysis/blob/main/data/N_2001-01-01_2021-11-11.csv)

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

### Data

### Plot

### Conclusion

## Further Ideas

-shipping rates

-flares

-[Global Conflict Risk Index (GCRI)](https://op.europa.eu/de/publication-detail/-/publication/1c121597-07cc-11e8-b8f5-01aa75ed71a1/language-en) combined with countries production
