# gas_market_analysis
Various attempts to predict the price of natural gas

## Reserves
###Intro
###Data
###Plot

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
![](https://github.com/ambader/gas_market_analysis/blob/main/img/price_res_corr.png?raw=true)
###Conclusion
