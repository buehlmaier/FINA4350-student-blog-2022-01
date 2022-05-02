---
Title: The Beginnings - Data Planning, Sourcing and Scraping (Group NLPQuartet) 
Date: 2022-04-14 10:45
Category: Progress Report
---

By Group "NLPQuartet," written by Kwan Wing Ching and Kwok Tsun Hei

Our project aims to investigate whether stock features react more towards news sentiment or social media sentiment, thus determine the predictability of these media channels on stock performance to improve data-driven decision making for investors. 

![Picture showing news]({static}/images/group-NLPQuartet-News.jpg)



As our very first blog post, we'd like to share our data collection journey. 

## Reflecting on our Strategy

We first established data standards, which outline the data items needed and how they should be collected. Originally, we had a lot of discussion as we pondered how to ensure the credibility of the news source, and how to select leading news providers in order to minimize reoccurence of articles that are republished. Then, we debated if that by itself is an issue because republished articles reflect the weight and impact of said articles on the Internet, etc. 

However, limited public news database access rendered a lot of these conceptual and theoretical considerations futile. Given Selenium access restrictions on a lot of sites, we faced many obstacles in the scraping process, so we finally resorted to using only Reuters. It is unfortunate that we could not base our study on more news sources; Despite that, this process has taught us that it isn't until we ‘get our hands dirty’ with the data mining process that we realize the quality and completeness of data that we’re dealt with. 

## Scrape Financial News Data

When we were doing web scraping on Reuters using selenium, we found that sometimes the HTML elements were not stored into the list we prepared. It is because there may be a connection issue on the webpage. To solve this issue, we set the time.sleep as well as using a for loop to perform 3 attempts on storing the HTML elements, in order to get every HTML element we need on every page.


```python
tickers= ['AAPL', 'MSFT', 'TSLA', 'AMZN', 'ATVI', 'NVDA', 'FB', 'UBER', 'V', 'MA', 'AVGO', 'CSCO', 'ADBE', 'CRM', 'AMD', 'INTC', 'NFLX']

for ticker in tickers: 
    searchFormat = 'https://www.reuters.com/site-search/?query=\
    {}&offset={}&sort=newest&date=past_year'  
    newsLinks = []

    for i in range(1,70):
        try: 
            driver.get(searchFormat.format(ticker, i*10-10))
            time.sleep(1) 
```

## Scrape Stock Data
Stock historical data scraping is perhaps the simplest of all data sourcing. We originally experimented with Selenium and BeautifulSoup to parse html from Yahoo Finance stock data by looping through the tickers. However, the process was slow and there was no straightforward and reproducible way to filter the months. Therefore, after some trials, we came across the yfinance package (an open-source tool that uses Yahoo's publicly available APIs) which is a much more efficient tool. It also allows us to fetch data by interval e.g. 60 minutes, which is potentially valuable to this project. Breaking news or tweets made by influencial account users can have a real-time impact on stock reactions, which would be insightful for us to capture. Therefore, considering the potential scalability, we decided to utilize this package. 


```python
for ticker in tickers:
#Get the data for the stock each stock
    data = yf.download(ticker,'2021-09-01','2022-03-01')
    data['Ticker'] = ticker
    data.to_csv('Price_'+ str(ticker) + '.csv')
```

## Scrape Twitter Data

Meanwhile, data scraping on twitter was unexpectedly tortuous. To begin, we tried some of the widely-used methods such as Twitter API, Twint and Tweepy. However, most of them impose download limits on free API key, rendering the database inaccessible. We then settled with snscrape, but the amount of Tweets scraped seems to be suspiciously few. To illustrate, even for popular tickers like TSLA, only 50-150 tweets are scraped daily, among which a lot appear to be in foreign languages and bot tweets. The other stocks recorded 15 tweets on a daily average.

```python
for i in tickers:
    #Establish search term ${Tciker} and #{Ticker}
    search_term = '#{ticker} ${ticker}'.format(ticker = i)

    # Using OS library to call CLI commands in Python
    snscrape_code = "snscrape --jsonl --since 2021-09-01 twitter-search \"{x} until:2022-03-01\" > {ticker}_tweetps.json".format(x = search_term, ticker = i)
    os.system(snscrape_code)
```
