---
Title: Data Scraping (Group Nebula)
Date: 2022-04-15
Category: Progress Report
---

By Group "Nebula"

## Team Nebula | Blog 1: Data Scraping

### Introduction
We hope to attain two goals in this project: 1) to create a model to predict the relationship between Form 10-K and the stock market performance on the release day, and 2) to compare the performance of different pre-processing techniques in NLP (Bag-of-Words and Sentiment Analysis) in a financial context.

In this blog post, we will detail our data scraping process in two parts: 1) scraping the text data (ie. Item 7 in Form 10Ks of our target companies), and 2) scraping the numerical data from Yahoo Finance.

### Textual Data Scraping - Christy
Scraping the textual data consists of two main tasks: finding the link to the desired Form 10-Ks, and extracting Item 7 from them.

The first task is easy with the use of `sec_api`: submit a request for the most recent Form 10-Ks for a given stock ticker, and repeat the process for each of our target stocks. Although we are only looking at data from the past 20 years, I have made the request for 25 reports instead to be safe because the API also returns non-10K forms (ie. amendments to Form 10Ks) occasionally, resulting in less than the requested amount of _actual_ Form 10-Ks.
```py
from sec_api import QueryApi
api_key = '' # replace this part with your API key, obtained from https://sec-api.io/
stock = 'AAPL' # this is declared earlier in a for loop, but added in for sake of demonstration
queryApi = QueryApi(api_key=api_key)
query = {
    "query": {
        "query_string": {
            "query": f"formType:\"10-K\" AND ticker:{stock}"
        }
    },
    "from": "0",
    "size": "25",

filings = queryApi.get_filings(query)['filings']

urls_for_stock = []
for filing in filings:
    urls_for_stock.append({'date': filing['filedAt'], 'url': filing['linkToFilingDetails']})
```
The query returns a dictionary object, and a list of relevant filings is found under the 'filings' key. I then save the `filedAt` (date) and `linkToFilingDetails` (url) attributes of each filing into a list, to be used in the second step.

The second step, to extract the Item 7 text using the url to the 10-K filing, was _extremely_ time-consuming. `sec_api` also had a function to extract individual sections given a url, but each pdf would take up one query, and the free tier of `sec_api` only allows for 100 queries (QueryApi used in the previous part only involved 1 query _per_ stock, which totalled 20 queries only). As such, we had to do it ourselves :')

Getting the _text_ given a url is not complicated, because luckily for us, the Form 10-Ks are in .htm format and the text could be easily extracted with requests and beautifulsoup's `get_text()`:
```py
import requests
from bs4 import BeautifulSoup

# header needed to declare usage / contact information as required by SEC EDGAR
headers = {'user-agent': '<contact information redacted> HKU student, for use in a course'}

url = 'https://www.sec.gov/Archives/edgar/data/320193/000032019318000145/a10-k20189292018.htm'
text = requests.get(url, headers=headers).text
bs = BeautifulSoup(text, features='lxml')
text = bs.get_text()
```
_Extracting Item 7_, however, was an entirely different matter. I originally hypothesized that it would be an easy job: all I had to do was to do a string search for "Item 7.", chop away everything before it and everything after the beginning of the next section ("Item 7A", which could easily be located with a string search). Section titles in Form 10-Ks are very standardized, so it shouldn't be a problem, _right_?
```py
searchterms = ('Item 7.', 'Item 7A.', 'Item 8.')

# get everything after 'Item 7.' and before 'Item 7A.'
text = text.split(searchterms[0])[1].split(searchterms[1])[0]

# make sure item 8 isn't attached (ie. if Item 7A doesn't exist)
if searchterms[2] in text:
    text = text.split(searchterms[2])[0]

```

I was _(extremely) wrong_. After looking at a couple of Form 10-Ks, I discovered that:
1. 'Item 7.' can appear in a lot of places, not just right before the actual MD&A section. The most common example of this is 'Item 7.' appearing in the table of contents as well.
2. there are slight variations of 'Item 7.' being used by different companies: "Item 7.", "ITEM 7.", "Item7.", "Item 07.", "Item 7--", "ITEM 7    \n:' are a couple of examples.

I dealt with the first issue by adding an extra check to make sure I have not accidentally extracted the 'Item 7.' in the table of contents. There is not much text between 'Item 7.' and 'Item 7A.' in the table of contents, so I could simply check for the length of the extracted text, and search for the next occurence if its length is beneath a certain threshold:
```py
if len(text) < 500: 
    text = bs.get_text()
    # get everything after the second 'Item 7.' and before 'Item 7A.' shows up again
    text = text.split(searchterms[0])[2].split(searchterms[1])[0]
```
I initially tried to deal with the second issue by brute force and switching the search terms based on string matches:
```py
searchterms = ('Item 7.', 'Item 7A.', 'Item 8.')
if 'Item 7--' in text: searchterms = ('Item 7--', 'Item 7A--', 'Item 8--')
if 'Item 7a.' in text: searchterms = ('Item 7.', 'Item 7a.', 'Item 8.')
if 'Item7.' in text: searchterms = ('Item7.', 'Item7A.', 'Item8.')
if 'ITEM 7.' in text: searchterms = ('ITEM 7.', 'ITEM 7A.', 'ITEM 8.')
if 'ITEM07.' in text: searchterms = ('ITEM07.', 'ITEM07A.', 'ITEM08.')
if 'ITEM7.' in text: searchterms = ('ITEM7.', 'ITEM7A.', 'ITEM8.')
if 'ITEM\n        7.' in text: searchterms = ('ITEM\n        7.', 'ITEM\n      7A.', 'ITEM\n        8.')
```
This approach had two major issues:
1. I found that there are too many combinations of irregularities to list them all out after going through more Form 10-Ks.
2. There are Form 10-Ks that would use at least two different types of searchterms (ie. use 'Item 7.' in the table of contents, but 'Item 7--' in the actual section header)

To overcome this, I opted to use a regular expression search instead to determine my searchterms:
```py
import re
first = re.findall('item *\n* *0?7\.', text, flags=re.I)
second = re.findall('item *\n* *0?7a\.', text, flags=re.I)
third = re.findall('item *\n* *0?8\.', text, flags=re.I)

# will explain + show remove_duplicates() below
first = remove_duplicates(first)
second = remove_duplicates(second)
third = remove_duplicates(third)

if len(first) == len(second) and len(second) == len(third):
    searchterms = [(first[i], second[i], third[i]) for i in range(len(first))]
    # sample searchterms: [('Item 7.', 'Item 7a.', 'Item 8.'), ('Item 7--', 'Item 7a--', 'Item 8--')]
```
`re.findall()` returns all matches of the regex search (ie. `['Item 7.', 'Item 7--']`) in the case where multiple types of searchterms show up. Searchterms is also now a list that stores different sets of search terms (ie. `[('Item 7.', 'Item 7a.', 'Item 8.'), ('Item 7--', 'Item 7a--', 'Item 8--')]`, and attempts at extracting Item 7 will be made using each set of search terms. 

"sets" of search terms are paired based on their order of appearance (a reasonable assumption: in the aforementioned case, "Item 7/7a/8." in the table of contents would always appear earlier than "Item 7/7a/8.--"). I noticed that some formats of search terms appear more than once (ie. `first = ['Item 7.', 'Item 7.', 'Item 7--']`), which would get in the way of pairing the correct search terms. To solve this, I had to remove duplicate entries first:
```py
# remove duplicates from first/second/third while preserving order
def remove_duplicates(keywords): 
    # converting keywords into a set and then back to list *would* remove duplicates, but not preserve the order :(
    seen = set()
    result = []
    for keyword in keywords:
        if keyword not in seen:
            seen.add(keyword)
            result.append(keyword)
    return result
```

However, my method of pairing searchterms into sets fails when one term has more variations than others: ie. `re.findall()` returns different numbers of searchterms for Item 7 (first), Item 7a (second), and Item 8 (third):
```
[Console]
first: ['ITEM 7.', 'ITEM7.']
second: ['ITEM 7A.', 'ITEM7A.']
third: ['ITEM 8.', 'Item 8.', 'ITEM8.', 'Item8.']
>> searchterms = [('ITEM 7.', 'ITEM 7A.', 'ITEM 8.'), ('ITEM7.', 'ITEM7A.', 'ITEM8.')]
```
There is very likely a more elegant / repeatable solution to this, but given the time constraint I chose to put a breakpoint if such a case is encountered and manually provide the sets of searchterms for the program, since these cases are sparse. Otherwise, I would have to make my code identify which terms belong to a single "set" of searchterms, which would likely require a lot of fine-tuning.

There are also Form 10-Ks that are very weird in format (such as [Costco's 2004 form](https://www.sec.gov/Archives/edgar/data/909832/000119312511271844/d203874d10k.htm), where 'Item 7--' appears on top of every page in the section as a header, or [Comcast's 2018 form](https://www.sec.gov/Archives/edgar/data/902739/000116669119000005/cmcsa-12312018x10k.htm), where there is special formatting and titles, like 'Item 7.', do not show up at all). Due to the time constraint, I have opted to not include these documents instead.

There are other minor issues that I have also tackled (ie. skipping past non form-10Ks, removing special characters that would get in the way of detecting 'Item 7', error handling etc), but not addressed here. My code has been modified / truncated to make it easier to follow.

As mentioned above, my code, regrettably, will not work on _all_ Form 10-Ks, and developing a truly comprehensive section extractor would likely involve a lot of work and testing. I believe the major takeaway from this is that real-world data is messy, even if you think they are already in a standardised format :')

### Financial Data Data Scraping - Eva
As we would like to compare the stock performance before and after the release of the Form 10-Ks, we need to use its historical financial data to perform the calculations and conduct further analysis. 

Based on the CSV that contains the data of stock names and Form 10-Ks release dates, the historical data of the 20 NASDAQ stocks on the release date and the day previous were extracted by using the Python package pandas_datareader.

```py
import pandas_datareader as pdr
data = pdr.get_data_yahoo(ticker, start_date, end_date) #start_date and end_date both equals the release date
```

However, the program did not run smoothly due to no matching data points. After checking the resource file, I realised some datapoints did not exist in Yahoo Finance. In light of this, I manually removed those datapoints, and amended the date of those that did not have their reports released on a business day to the next business day. 

I then tried to run the program again and successfully got an output. Yet, the number of rows in the output differed from the expected number. After investigation, the problem was due to the gap between the business days. If the previous business day has a gap of a day or more with the release date, the program will not be able to include both days' data in the dataframe. And I had no idea how could I fix this problem.

So, I decided to change the Python package I used for extracting the data. 

```py
import yfinance as yf
data = yf.download(ticker, start = start_date, end = end_date) #start_date = release day - n business day; end_date = release day + 1 business day
```

This new version of code gave me an output with incorrect number of results again. So, I checked the csv generated from this program and found out that the previous business day cannot be captured simply with start_date = release day - 1 business day, as there may be a gap with one or more days between the business days. To solve this, I created a for loop to check if the length of data extracted equals to 2 (which means both days' data were captured). If it does not, a while loop starts and tries difference business day gaps until there is a length of 2. 

```py
if len(stock_data) == 2:
        stock_data = pd.DataFrame(data)
        df = df.append(stock_data)
    elif len(stock_data) == 1:
        i = 1
        while len(stock_data) != 2:
            start_date = end_date_obj-BDay(i)+datetime.timedelta(days=1)
            start_date = str(start_date)[0:10]
            data = yf.download(ticker, start=start_date, end=end_date)
            stock_data = pd.DataFrame(data)
            i += 1
        df = df.append(stock_data)
```

This amendment successfully gave me the expected number of output. 

Another thing that I had to do is to add the Ticker name into the dataframe, as this information was not included when the data were extracted. Therefore, I created a blank list in the beginning and append the ticker name into the list according to the count of data for each stock. The list was then added to the dataframe as a new column "Ticker".

```py
for j in range(len(df)-count):
        df_ticker.append(ticker)
    count = len(df)

df["Ticker"] = df_ticker
```

After producing the dataframe, a CSV file was generated for the other group mates to work on the next step.

```py
print(df)
df.to_csv("historical_data_new.csv")
```

### Summary
This is basically what we did in the first stage: data scraping. 

There were some hiccups in the progress, so we were not able to complete this stage in the time we had proposed. We reflected on our time management and hope we can meet the deadline for the second stage.
