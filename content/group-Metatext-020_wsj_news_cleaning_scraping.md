---
Title: WSJ Scrapping and Pre-processing (Group Metatext)
Date: 2022-04-23 23:00
Category: Progress Report
---

By Group "Metatext," written by Rex Chan

Important: This blog post is of academic nature and for educational
purpose only. Group Metatext pledges to follow the download
regulations of databases such as Factiva.


## Intro & Objective

Our project analyzes whether Federal Reserves' FOMC documents and news articles could possibly predict where interest rates are heading. This is a critical question, especially given that Fed has suddenly turned hawkish to curb inflation. Everyone wants to know where interest rates might be in the near future. 

## Why News Article?

Naturally, when one is to predict Fed's decision, he delve into Fed's document, namely press conference, minutes, etc. (Stay tuned for my other group-mate detailing how to scrap and preprocess them). Fed's document inclines to plain fact reporting. News articles however, offer more subjective commentary on the cause and the impact of Fed's decision, which might offer more valuable information regarding how Wall Street perceive Fed's action. At the end of the day, Fed only manipulate the the policy rate, it is up to the Wall Street to move the market and propagate the effect of Fed's move to every corner of the financial market.

From a project perspective, news article can been seen as a benchmark where one could compare Fed's official document against news article. 

We decided to opt for articles on Wall Street Journal. Reports from WSJ offers a delicate balance between depth and breath versus, say Bloomberg which focuses on speed and breath of reports. 

## How To Scrap News Articles?

***Libraries and Modules Used:***

We first import the following libraries.

```python
from selenium import webdriver
import time, os
import re
import math
```

're' (regular expressions) is a package that specialize in manipulating and interact with strings/text objects.

'selenium webdriver' is a package that enable the code to control the web browser (such as Google Chrome and Firefox) as if we are manually browsing on the browser. Whenever we are working with dynamic websites that are heavily java-scriptted, we can use 'webdriver' to monitor how javascript changes HTML. 

We will be using Dow Jones Factiva to scrap news. Factiva is a paid-service with API access to for institutions. However, we can access Factiva via the link available at HKU Library using HKU credentials without API access. Therefore, we create a 'synthetic API' using the webdriver module to access Factiva shown as follows: 

```python
webdriver_dir = '/Users/chanmingho/Downloads/chromedriver' #supply local directory of 'webdriver'
download_dir = '/Users/chanmingho/Downloads/FINA4350_GroupProject'
#supply local directory of where output files should be downloaded

#check if download_dir has alraedy been created, if not create one
isExist = os.path.exists(download_dir)
if not isExist:
    os.makedirs(download_dir)
os.chdir(download_dir)

#Input URL to login Factiva from HKU Library
hku_login = 'https://julac-hku-a.alma.exlibrisgroup.com/view/action/uresolver.do;jsessionid=3785A9C82A112175676195A7044CA0A4.app03.ap01.prod.alma.dc05.hosted.exlibrisgroup.com:1801?operation=resolveService&package_service_id=17357882830003414&institutionId=3414&customerId=3405'

#Input HKU Credentials
login = {'username': 'UID', 'password': 'PIN'}

#Input Data Range
data_range = {'Start': '2017-01-01', 'End': '2022-04-11'}

#Browse Factiva
driver = webdriver.Chrome(webdriver_dir)
driver.get(hku_login)
time.sleep(3)

#Input UID and password
driver.find_element_by_xpath('//*[@id="hkulauth"]/p[2]/input').send_keys(login['username'])
driver.find_element_by_xpath('//*[@id="hkulauth"]/p[3]/input').send_keys(login['password'])
driver.find_element_by_xpath('//*[@id="hkulauth"]/p[6]/input[1]').click()
time.sleep(25)
```

We arrive at the following Factiva search page:

![Factiva search page]({static}/images/group-Metatext-020_factiva.png)

We proceed to input the appropriate filters shown as follows. We only look for news that are in English, the subject of the article is related to interest rates and the company is Fed with the desired date range, which in our case, is anytime after 1 Jan 2017.
 
```python
#Filter news to desired date range
driver.find_element_by_xpath('//*[@id="dr"]/option[10]').click()
driver.find_element_by_xpath('//*[@id="frd"]').send_keys(re.split('-', data_range['Start'])[2])
driver.find_element_by_xpath('//*[@id="frm"]').send_keys(re.split('-', data_range['Start'])[1])  
driver.find_element_by_xpath('//*[@id="fry"]').send_keys(re.split('-', data_range['Start'])[0])  
driver.find_element_by_xpath('//*[@id="tod"]').send_keys(re.split('-', data_range['End'])[2])
driver.find_element_by_xpath('//*[@id="tom"]').send_keys(re.split('-', data_range['End'])[1])  
driver.find_element_by_xpath('//*[@id="toy"]').send_keys(re.split('-', data_range['End'])[0])

driver.find_element_by_xpath('//*[@id="isrd"]/option[3]').click()

#Filter news agency to only WSJ
driver.find_element_by_xpath('//*[@id="scTab"]/div[1]').click()
driver.find_element_by_xpath('//*[@id="scLst"]/div/ul/li/div/div/span').click()
driver.find_element_by_xpath('//*[@id="scpillscontextmenu"]/div/div[2]/span').click()
driver.find_element_by_xpath('//*[@id="scTxt"]').send_keys('The Wall Street Journal - All Sources')
time.sleep(3)
driver.find_element_by_xpath('/html/body/div[13]/table/tbody/tr/td/div').click()

#Filter 'company' to Fed
driver.find_element_by_xpath('//*[@id="coTab"]/div[1]').click()
driver.find_element_by_xpath('//*[@id="coTxt"]').send_keys('Board of Governors of the Federal Reserve System')
time.sleep(3)
driver.find_element_by_xpath('/html/body/div[14]/table/tbody/tr/td/div').click()

#Filter subject to interest rates
driver.find_element_by_xpath('//*[@id="nsTab"]/div[1]').click()
driver.find_element_by_xpath('//*[@id="nsTxt"]').send_keys('Interest Rates')
time.sleep(3)
driver.find_element_by_xpath('/html/body/div[15]/table/tbody/tr/td/div').click()

#Filter country to only US
driver.find_element_by_xpath('//*[@id="reTab"]/div[1]').click()
driver.find_element_by_xpath('//*[@id="reTxt"]').send_keys('United States')
time.sleep(3)
driver.find_element_by_xpath('/html/body/div[16]/table/tbody/tr[1]/td/div').click()

driver.find_element_by_xpath('//*[@id="btnSearchBottom"]').click()
time.sleep(7)
```

We arrive at the following search result page:

![Factiva search page]({static}/images/group-Metatext-020_factiva_search_result.png)

We proceed to download search results page by page as PDFs into local directories. 

```python
news_count = int(driver.find_element_by_xpath('//*[@id="headlineHeader33"]/table/tbody/tr/td/span[2]').get_attribute('data-hits'))
page_count = math.floor(news_count/100)
for page in range (3, page_count+1):
    driver.find_element_by_xpath('//*[@id="selectAll"]/input').click()
    driver.find_element_by_xpath('//*[@id="listMenuRoot"]/li[6]/a').click()
    driver.find_element_by_xpath('//*[@id="listMenu-id-4"]/li[2]/a').click()
    driver.find_element_by_xpath('//*[@id="clearAll"]/input').click()
    driver.find_element_by_xpath('//*[@id="headlineHeader33"]/table/tbody/tr/td/a').click()
    time.sleep(30)
```

## How To Preprocess News Articles?

A typical news article scrapped from Factiva is shown as follows:

![A sample of WSJ news scrapped from Factiva]({static}/images/group-Metatext-020_sample_of_news.jpeg)

All we need from news articles is author, date, title and the main text.

We first upload the PDFs into Google Drive under a specified directory. Then we read text from PDFs into a variable.

```python
all_news = ""
news_for_analysis = []

for filename in os.listdir("/content/drive/My Drive/FINA4350/Article"):
  if filename.endswith(".pdf"): 
    with fitz.open(filename) as pdf:
      for page in pdf:
        all_news += page.get_text()
```

Note that some articles are very lengthy, it is not possible to separate one news from another using simply page number. However, note that whenever each article ends, there is a 'Document *X*' pattern where *X* is a combination of numbers and characters. Hence, we ask Python to separate news whenever it detects such patterns.

```python
all_news_splitted = all_news.split('Document ')

for each_news in all_news_splitted:
  each_news = each_news[each_news.find('\n',1):] 
```

Upon closer look, we observe that Factiva search result also outputs non-news article such as interview transcript, podcast, preview of news, amendments etc. Also, last page of PDFs also contains a 'search summary' which is irrelavant. Any text with word count under 500 words are unlikely to contain any meaningful content, or are non-news articles. The following code removes these texts.

```python
all_news_cleaned_filtered = []

  if (len(each_news)) > 500:
    if (("Search Summary" not in each_news) and ('WSJ Pro Central Banking,' not in each_news) and ('WSJ Podcast Minute Briefing' not in each_news) and ("WSJ Blogs" not in each_news) and \
        ("WSJ Podcast What's News" not in each_news) and ("Transcript:" not in each_news) and ("Federal Reserve Readies More Aggressive Taper of Bond-Buying as Focus Turns to Inflation" not in each_news) and \
        ("Corrections & Amplifications\nCorrections & Amplifications\n" not in each_news)):
      all_news_cleaned_filtered.append(each_news)
```

After filtering out non-news articles, we proceed to clean up each individual articles. At the first line of some articles, it contains name of the news category (e.g. Central Banking, WSJ Pro, etc.). Language, disclaimer, word count, page number, time of publishing, contact of author, , identifiers, etc. They all need to be stripped to obtain relevant information, with the help of the 're' library.

```python
for each_cleaned_filtered_news in all_news_cleaned_filtered:
    each_cleaned_filtered_news = each_cleaned_filtered_news.replace('License this article from Dow Jones Reprint Service', '').replace('Corrections & Amplifications', '')

    each_cleaned_filtered_news = re.sub('Copyright \d+ Dow Jones & Company, Inc. All Rights Reserved.', '', each_cleaned_filtered_news)
    each_cleaned_filtered_news = re.sub('Copyright ?(.*?), Inc.', '', each_cleaned_filtered_news)
    each_cleaned_filtered_news = re.sub('\d+ words|\d+,\d+ words', '', each_cleaned_filtered_news)
    each_cleaned_filtered_news = re.sub('Page?(.*?)All rights reserved.','', each_cleaned_filtered_news)
    each_cleaned_filtered_news = re.sub('\d+:\d+', '', each_cleaned_filtered_news)

    each_cleaned_filtered_news = re.sub("WSJ Pro\n|The Outlook\n|Economy\n|Heard on the Street|Markets\n|PERSONAL JOURNAL|Central Banks\n\w+\n|Central Banks\n|The Wall Street Journal Online\n| \
                                        Today's Markets\nMarkets|Heard on the Street\nMarkets|Journal Reports|US\n|Finance\n|U.S. Economy\n|Markets Main\n|B\d+\n|A\d+\n|J\n", '', each_cleaned_filtered_news)

    each_cleaned_filtered_news = each_cleaned_filtered_news.replace("WSJ Pro Central Banking\n","").replace("CFO Journal\n","").replace("C Suite\n","").replace("The Wall Street Journal",""). \
                                                            replace("English\n","").replace("The Wall Street Journal Online\n","").replace("WSJO\n","").replace("RSTPROCB\n","").replace("World\n","") \
                                                            .replace("Opinion\n", "").replace("Commentary (U.S.)\nOpinion\n", "").replace("Commentary (U.S.)", "").replace("[Financial Analysis and Commentary]", '')

    if each_cleaned_filtered_news.count('@') == 1:
      each_cleaned_filtered_news = re.sub('Write to?(.*?)com', '', each_cleaned_filtered_news , flags=re.DOTALL)
    elif each_cleaned_filtered_news.count('@') == 2:
      each_cleaned_filtered_news = re.sub('Write to?(.*?)and?(.*?)com', '', each_cleaned_filtered_news , flags=re.DOTALL)
    elif each_cleaned_filtered_news.count('@') == 3:
      each_cleaned_filtered_news = re.sub('Write to?(.*?),?(.*?)and?(.*?)com', '', each_cleaned_filtered_news , flags=re.DOTALL)

    each_cleaned_filtered_news = each_cleaned_filtered_news.strip()
```

Finally, the text now contains only information we need and nothing else. The following code extract author, date, title and main text from resulting files. Note that 'Review & Outlook' series has no author because it is written by the Editorial. We simply assign 'WSJ' as author to these pieces.

```python
    if 'REVIEW & OUTLOOK (Editorial)' not in each_cleaned_filtered_news:
      title = each_cleaned_filtered_news[:each_cleaned_filtered_news.find('By')-1].strip()
      try:
        author = each_cleaned_filtered_news[re.search('\nBy ', each_cleaned_filtered_news).end():re.search("\n\d+", each_cleaned_filtered_news).start()-1]
      except:
        author = 'WSJ'
    else:
      title = each_cleaned_filtered_news[re.search('(Editorial)', each_cleaned_filtered_news).end()+2:re.search("\n\d+", each_cleaned_filtered_news).start()-1]
      author = 'WSJ'
    
    date = each_cleaned_filtered_news[re.search("\n\d+", each_cleaned_filtered_news).start():re.search('20\d+\n',each_cleaned_filtered_news).end()-1]
    body = each_cleaned_filtered_news[re.search('20\d+\n',each_cleaned_filtered_news).end()-1:]

    news_for_analysis.append([title, author, date.strip(), body.strip()])

all_news = ""
```

As a sanity check, we remove any news that has missing author, title, date or main text, in case there are any articles that does not follow the example shown above.

```python
for element in news_for_analysis:
  if ((element[0] == '') or (element[1] == '') or (element[2] == '') or (element[3] == '')):
    news_for_analysis.remove(element)
```

## After Thoughts

After I found that pre-processing is such a grunt and tedious work, I have come to agree to the statement 'NLP is 95% pre-processing'. As an old saying goes, 'garbage in, garbage out'. It is of utmost importance to have quality input available for analysis. If non-body text pollutes the news articles, that would only output mediocre results.

While we have yet to derive at the results, I would personally doubt whether our analysis would have any predictive power. If computing textual probabilities of Fed's rate decision using Fed's document is that easy and straightforward, many would have simply arbitraged away the trading opportunity for risk-free profits already via fed fund rate futures or interest rate swaps. This means our work would be rendered meaningless. However, this is clearly not the case. Each strategist might be using different modules, assumptions, methodologies, etc to derive textual probabilities and come up with different results. Our group is no exception. 

FINA4350 is a great and interesting course where I learn a lot. 

