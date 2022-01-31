Title: Text Analytics
Date: 2021-12-03 17:20
Category: Progress Report

By Group "Super NLP"

In the previous blog, our team shows the progress we made in data
mining and preprocessing. This blog we will guide you through our
sentiment analysis toward XRP related tweets.

# Financial Data 

Before we can start the sentiment analysis, we found that part of the
financial data of XRP we loaded from Yahoo Finance was
missing. Specifically, the data on October 12, 2020 was missing in the
Yahoo Finance database. So we chose
[Investing.com](http://www.investing.com) to get the whole year data
of XRP and recalculated the return and 30 days volatility.

The code we use is as follows:
```python
import nltk
import pandas as pd
myvar = 8
DF = pd.read_csv('XRP-data.csv')
```
