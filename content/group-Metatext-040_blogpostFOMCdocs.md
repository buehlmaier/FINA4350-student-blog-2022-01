---
Title: Fed Watching - 2-year Treasuries Yield prediction utilizing FOMC & WSJ scraped data (Group Metatext)
Date: 2022-05-01 15:01
Category: Progress Report
---

By Group "Metatext"


## Intro & Objective

The objective of our project is to predict the 2 years Treasury Yield at the intersection of The Federal Open Markets Committee (FOMC) documents and Wall Street news Articles scraped data. Treasury Yield is a rate at which the US government borrows. It’s one of the most important rates in the context of the US economy as it impacts the monetary and financial condition including the broader economy such as employment, growth, and inflation. It has an indirect impact on the other interest rate measures. For example, it determines the rate for home, auto loans and credit loans. Also, investors are particularly curious about it because the stock market is strongly reactive to this rate. 

## Our thought processes

We decided to conduct our analysis using two different data sources: FOMC documents including statements, meeting minutes, press conferences and testimonials, and Wall Street News Articles in hopes of reaching relatively accurate prediction results. Here is why; news articles serve as a benchmark of comparison as it provides more detailed and subjective interpretations of feds documents which opens room for more comprehension. Secondly, two datasets are advantageous in terms of providing more useful information that can potentially increase the probability of accurate prediction.

We have already posted our workflow for scrapping News Articles. This blog post contains our workflow for FOMC documents. We have scraped all four documents from the FOMC website.

## Data Collection
 
How to scrape FOMC data?
 
*Important libraries & Modules  
```python
from  datetime import date
import re
import threading
import pickle
import sys
import os
import requests
from bs4 import BeautifulSoup
import numpy as np
import pandas as pd
from abc import ABCMeta, abstractmethod
```
# Our Approach 

First, we create two base files. The first file is FOMCBase.py, which receives the scraped data and converts it into a text file. The second file is FOMCGetData.py, which specifies the period of data and dump the scraped data into a pickle file. Next, we created a folder named fomc_get_data, which contains all the code files to scrape the specific type of data from the FOMC website such as FomcMinutes.py, FomcPresConfScript.py, FomcTestimony.py, FomcStatement.py. The final output will be a folder named FOMC which contains the pickle and csv files of all the data scraped. 
 


![ This is an example of the statement]({static}/images/group-Metatext-sdatafram1.png)

## Data Cleaning, Reshaping, Preprocessing

After scraping the data from the FOMC website. We noticed there are many unnecessary words which will affect our analysis. Thus, we preprocessed the scraped data in three ways:

1)	Remove the text sections that has less than 50 words as it is unlikely to have insightful information 
2)	Split the text sections that has more than 200 words 
3)	Remove the text sections that does not contain at least 2 keywords ('rate', 'rates', 'federal fund', 'outlook', 'forecast', 'employ', 'economy')

Before we proceed with the data preprocessing, we cleaned and reshaped the data by adding additional columns to the current data frame such as word count and next meeting date. We also added a “text_section” column which splits the text according to [SECTION] and “org_text” which is a copy of the original raw data scraped. The “org_text” column acts as a “back up” and comparison with our changes to the data. 

Data preprocessing is performed for all 4 types of data separately. We then aggregated the data according to the preprocessing technique. 

The final output will be three files namely, text_no_split.pickle, text_split_200.pickle and text_keyword.pickle. 

![This is an example of the statement]({static}/images/group-Metatext-sdataframe.png)

## Analysis and conclusion 

To analyze, we run a thorough sentiment analysis on both scraped datasets. We found their respective polarity and subjectivity plots. Our interpretation of the plots reveals that FOMC documents are better indicators of upcoming rate changes compared to Wallstreet News Articles. Thus, we suggest interested stakeholders refer to FOMC documents for better analysis and prediction of rates.  