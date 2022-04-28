---
Title: Data Pre-Processing (Group Nebula)
Date: 2022-04-20
Category: Progress Report
---

By Group "Nebula"

## Team Nebula | Blog 2: Data Pre-Processing

### Introduction
In this blog post, we will detail our data pre-processing process for cleaning the text data and calculating the stock performance.

### Textual Data Cleaning
As the data were directly collected from HTML, there were items that we do not need for the analysis, like symbols and numbers that had no textual meaning e.g. % | # etc, and unusual patterns within the text e.g. the "!." in "Hello World!.". After removing the unnecessary items, the text were joined together into one paragraph, instead of multiple ones in a text file.

```python
import re
pattern = r"\b\w*[A-z'’\-\&]\w*\b[.,]?|\b\w*[.]\w*\b[.,]?"
with open(file_name,"r", encoding='utf-8') as f:
    txt = f.read()
    x = re.findall(pattern,txt)
with open("out.txt","w") as f:
    f.write(' '.join(x))
```

### Stock Performance Calculation
To analyse the stock performance, we first extracted the beta of each stock from yfinance.

```python
import yfinance as yf
for ticker in ticker_list:
    beta = yf.Ticker(ticker).info['beta']
```

We then merged all the useful data from the CSV generated in the previous stage, beta (from previous step), and NASDAQ historical data into one dataframe for further analysis. 

```python
processed_historical_data = historical_data.copy()

processed_historical_data = processed_historical_data.drop(['Open', 'High', 'Low', 'Adj Close', 'Volume'], axis=1)

processed_historical_data['Nasdaq Close'] = ixic_close
processed_historical_data['Beta'] = stock_beta

processed_historical_data = processed_historical_data[['Date', 'Ticker', 'Beta', 'Close', 'Nasdaq Close']]
```

To calculate the performance of each stock at each time point, we adopted the equation:
>Expected Return = Beta * Market Return

To analyse the performance of each data, we compared the actual return to the expected return. If the expected return is larger than the actual return, a negative performance was identified and returned as "0". Otherwise, it would be classified as a positive performance and returned as "1".

```python
performance = []

for i in range(0, len(processed_historical_data), 2):
    mkt_old = processed_historical_data.loc[i]['Nasdaq Close']
    mkt_new = processed_historical_data.loc[i+1]['Nasdaq Close']
    mkt_return = (mkt_new-mkt_old)/mkt_old
    
    stk_old = processed_historical_data.loc[i]['Close']
    stk_new = processed_historical_data.loc[i+1]['Close']
    stk_return = (stk_new-stk_old)/stk_old
    
    stk_beta = processed_historical_data.loc[i]['Beta']
    
    expected_stk_return = mkt_return*stk_beta
    
    stk_perform = stk_return-expected_stk_return
    
    performance.append(None)
    
    if(stk_perform < 0):
        performance.append(0)
    else:
        performance.append(1)
```

### Combining Pre-processed Textual and Stock Data
We first scan though the directory that contains all text data you scrapped previously.

```python
textdata_dir = './textdata'

textdata_paths = [ os.path.join(textdata_dir, file_name) for file_name in  os.listdir(textdata_dir) if file_name.endswith('.txt') ]
```

After carrying out the steps mentioned above, we joined the cleaned report with the corresponding stock and report released date, such that we can easily call the data in the later stage. At first, we faced some problems as non `.txt` files were also included. To solve this problem, we added `if file_name.endswith('.txt')` when we scan through the directory.

```python
text_data = pd.DataFrame(columns=['Date','Ticker','Text'])

for path in textdata_paths:
    f = open(path,'r')
    text = f.read()
    f.close()
    
    date = re.search('_(.+?).txt', path).group(1)
    ticker = re.search('./cleanedtextdata/(.+?)_', path).group(1)
    
    row = [date, ticker, text]
    
    df_row = pd.DataFrame([row], columns=['Date','Ticker','Text'])
    text_data = text_data.append(df_row, ignore_index=True)

text_data = text_data.sort_values(['Ticker', 'Date'])
text_data
```
