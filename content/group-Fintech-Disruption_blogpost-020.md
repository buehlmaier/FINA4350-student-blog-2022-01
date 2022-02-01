Title: [Demo Blog Post] Fed Watching
Date: 2022-01-25 13:58
Category: Summary of Findings

By Group "Fintech Disruption"

One of the two main objectives of this project is to find some pattern
between the speeches made by FED personnel throughout the year and the
impact, if any, they have on the FED Fund Rate.

# Background Information on the Fed

The [Fed](https://www.federalreserve.gov/) has mainly three appointed
positions&mdash;the Chair, the Vice Chair and the Governor. Out of
these three positions the Chair holds the highest stature and is often
viewed as the main answerable authority of the FED. The Vice Chair and
Governors are other members in the seven board committee appointed by
the President of the United States.

As a famous hedge fund manager once said:
>Fed watching is a great tool to make money. I have been making all my
>gazillions using this technique.

The code we use is as follows:
```python
import gensim
import pandas as pd
DF = pd.read_csv('FED-data.csv')
```
