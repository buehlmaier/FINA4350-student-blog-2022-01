---
Title: Research Overview (Group Snapshot)
Date: 2022-05-02 01:12
Category: Progress Report
---

By Group "Snapshot", written by Alessia Ciocanea

Hi everyone. In this article, I will summarise the existing literature that inspired our topic and model.

## Introduction and Set-up 

We chose to explore the impact of social media sentiment on bitcoin volatility. Despite high volumes, the bitcoin market is still in its infancy, and market participants are still debating whether the cryptocurrency is a store of value, an asset class, a means of exchange, or a combination of the three. As bitcoin (and cryptocurrencies in general) can't be valued using traditional valuation techniques such as cashflows for stocks or industrial uses for precious metals, we had to explore alternative methods of measuring the price and volatility of bitcoin. It is exactly because bitcoin price formation is not understood that researchers theorize bitcoin is exposed to sentiment.

Inspired by Brown, (1999) we delved further into the relationship between sentiment and volatility: "If noise traders affect prices, the noisy signal is sentiment, and the risk they cause is volatility, then sentiment should be correlated with volatility". There are many proxies for sentiment, including market volume, survey-based indices and google trends data, all of which are considered in the literature. We chose to use natural language processing techniques on recent social media posts across two platforms, Twitter and Reddit. Our analysis therefore contributes to recent literature by combining the platforms and expanding the analysis to the lesser-used Reddit comments on bitcoin.


## Data

We surveyed the literature to gain insight into what data we should collect. As explained in a previous blogpost on the data collected, we ended up scraping hourly, rather than daily, data for our model. While this improved our random sample given the limitations of the code, it likely lead to an overfitted trend in the data, in the same way that weekly data would have smoothed out the effect of the sentiment-related regressors in our model.

Nevertheless, there is backing for our decision. For one, it could be argued that the short shelf-life and immediacy of tweets means that autocorrelations are most significant over a period shorter than a day, as social media users react and respond to each other. We found one other paper, Gao et al, (2021), which uses hourly data, arguing that this is a better mechanism to capture the momentum of sentiment on social media.

Other studies that used daily or weekly data likely capture less of this intraday momentum and interactions, but they look at volatility trends over larger time periods. Given we collected data for the period 1st August 2021 to 9th March 2022, hourly data is probably more appropriate in order to capture trends over this shorter period of time.

Our analysis can be extended along the lines of existing literature to analyse longer-term volatility trends, although that would likely require controlling for other market events that could affect volatility of all assets, including bitcoin. Moreover, we thought such an analysis would be less appropriate for this project because of the computational intensity of scraping more data. The scraping of the twitter data alone took 14 hours on the devices we have at our disposal.


## Model

Bukovina, J., Marticek, M., (2016) use a simple AR(1) model and observe a minor predictive power of sentiment to bitcoin volatility. Their main contribution is splitting volatility into a rational and irrational component based on the average number of transactions per block and the revenue associated with them. After 2015 however, this breakdown loses its descriptive power as the volatility of the transaction rate increases. As our data is from 2021 onwards, we therefore avoided this modelling split and chose to build on it by considering a GARCH model.

GARCH models are the most popular in the literature. The following papers utilise them: Lopez-Cabarcos et al., (2019); Guler, (2021); and Akbiyik et al., (2021). The latter finds that their AR-RV model performs better than the GARCH. However, due to the wider range of literature based on GARCH models, we employed only this method so that we could compare and contextualise our results. Moreover, we considered that the GARCH model is sufficiently complex for the purposes of our analysis, insofar as it accounts for volatility clustering and heteroskedasticity-induced variance. The added external regressors were used to build on existing literature and ultimately lead to a better fit to our data than the simple GARCH(1,1) model. A possible extension is to use the EGARCH model, which is widely used in the literature in order to handle the non-negativity constraints of the GARCH.

## References

1. Ciaian, P., Rajcaniova, M. and Kancs, D. (2016) The Economics of Bitcoin Price Formation. Applied Economics, 48, 1799-1815.
2. Gregory W. Brown (1999) Volatility, Sentiment, and Noise Traders, Financial Analysts Journal, 55:2, 82-90
3. Derya Guler (2021) The Impact of Investor Sentiment on Bitcoin Returns and Conditional Volatilities during the Era of Covid-19, Journal of Behavioral Finance
4. Leopoldo Catania (2019) Bitcoin at High Frequency, Journal of Risk and Financial Management 12(1):36
5. Gao et al. (2021) Financial Twitter Sentiment on Bitcoin Return and High-Frequency Volatility, Virtual Economics Vol 4 Nr 1
6. Jaroslav Bukovina & Matus Marticek, 2016. Sentiment and Bitcoin Volatility, MENDELU Working Papers in Business and Economics 2016-58
7. Akbiyik et al.(2021) Ask Who, Not What: Bitcoin Volatility Forecasting with Twitter Data
8. Lopez-Cabarcos et al. (2019) Bitcoin volatility, stock market and investor sentiment. Are they connected?, Finance Research Letters, Volume 38-2021
