---
Title: Twitter Scraping (Group Snapshot)
Date: 2022-04-15 21:59
Category: Progress Report
---

By Group "Snapshot", written by Patrick van Ewijk

Dear readers, this is the first blog post of our group, and its regarding the scraping of the twitter data. 
The blog describes the progress of the scraping, the problems I ran into and the corresponding solutions.
As the question is whether the volatility of the bitcoin data can be predicted using the sentiment on twitter and reddit, historical data is of interest. Hence, for the twitter part we used the *snscrape* package. Further, we need *timedelta* (and later on *timezone*) from *datetime*

```python
import snscrape.modules.twitter as sntwitter
from datetime import timedelta, timezone
```

#  Our initial code
We wrote a function which is shown below in order to scrape tweets from twitter. First, the function is presented. Hereafter, problems we ran into will be discussed.
```python
def ScrapData(MaxLimit, Date_Start, Date_End, SearchQuery, filename=None):
    """
    Function that scrapes twitter data.
    
    Input:
    MaxLimit: Maximum number of tweets one wants to scrape during a day.
    Date_Start: Starting date. String in format: yyy-mm-dd
    Date_End: Ending date. String in format: yyy-mm-dd
    SearchQuery: What does one want to search on twitter?
    Optional Input:
    filename: Name of the file where one wants to write the output.
    Standard filename=None and the output is not stored.
    Output:
    dictionary with as keys the dates and
    as values a list of scraped tweets during the day.
    """
    Dates=pd.date_range(start=Date_Start,end=Date_End)
    Dates_TweetsondayDict={}

    for day in Dates:
        day_next=day+timedelta(days=1)
        #running search
        Query=f"{SearchQuery} since:{day.date()} until:{day_next.date()}"
        tweets_list2 = []
        for iteration,tweet in \
            enumerate(sntwitter.TwitterSearchScraper(Query).get_items()):
            if iteration>MaxLimit:
                break
            tweets_list2.append(tweet)
        Dates_TweetsondayDict[day]=tweets_list2
    if filename is not None:
        with open(filename, 'wb') as my_file:
            pic.dump(Dates_TweetsondayDict, my_file, pic.HIGHEST_PROTOCOL)        
    return Dates_TweetsondayDict
```
For technical reasons, we made the decision to choose `MaxLimit=6000`. Scraping the data took 14 hours. For a long time I was satisfied with the output of the function. Everything seemed to work.
However, later on I found out that this function was not optimal.

## Problems

**Problem 1: Not only tweets in English captured**

During the cleaning of the data, we found out that not only Tweets in English were collected; also tweets in Chinese and other languages were found. Of course, one could solve this by cleaning the data after the scraping. However, we preferred 'cleaner searching' over cleaning the searched data based on language afterwards.

**Problem 2: Collected tweets are not a random sample of all tweets during the day** 

It turned out that the function scraped only the last `MaxLimit` tweets of the day. In other words, in practice only tweets after 23.00 were captured. One could argue that the random sample assumption would be violated in this way. The information captured in tweets after 23.00 might be different from tweets during the rest of the day. Although this is not precisely bad, it is not what we wanted. In the optimal situation, we would like a sample of our tweets that represents the sentiment during the day as good as possible. 

# Improving our code

Solving **Problem 1** was relatively easy. By replacing `Query` by  
```python
Query=f"{SearchQuery} since_time:{date} until_time:{date_next} lang:en"
```
this problem is solved. Solving **Problem 2** was more difficult however.

Basically, I came up with two "solutions" for **Problem 2**.
* Increasing the `Maxlimit` such that Tweets over the whole day are captured.
* Searching hourly data, instead of daily data.

The first thought wouldn't be possible, because of technical constraints. The scraping would take , if one assumes a linear relation between execution time and #tweets, over 336 hours and the data files would be too large.

Hence, I implemented the second idea. However, when one searches on Twitter, one can only choose the day and not the time (see [twitter.com](https://twitter.com/search-advanced)). That is, using the combination `since_time: 01-08-2021 00:00:00` and `until_time  01-08-2021 00:01:00` does not work.

Luckily, searching on the web usually leads to a solution. It turned out that one can use the *unix* time as an argument in `since_time` and `until_time`.


This resulted in the following function. 

```python
def ScrapData(MaxLimit, Date_Start, Date_End, SearchQuery, filename=None):
    Dates=pd.date_range(start=Date_Start,end=Date_End, freq='h')
    Dates_TweetsondayDict={}

    for date in Dates:
        date_unix=int(date.replace(tzinfo=timezone.utc).timestamp())
        date_unix_next=int((date+timedelta(hours=1)).replace(tzinfo=timezone.utc).timestamp())
        #running search
        Query=f"{SearchQuery} since_time:{date_unix} until_time:{date_unix_next} lang:en"
        tweets_list2 = []
        for iteration,tweet in \
            enumerate(sntwitter.TwitterSearchScraper(Query).get_items()):
            if iteration>MaxLimit:
                break
            tweets_list2.append(tweet)
        Dates_TweetsondayDict[date_unix]=tweets_list2
    if filename is not None:
        with open(filename, 'wb') as my_file:
            pic.dump(Dates_TweetsondayDict, my_file, pic.HIGHEST_PROTOCOL)        
    return Dates_TweetsondayDict
```
Here we set `MaxLimit` 24 times as low as before. Further, we need the *timezone* to map our dates to the unix format.

## Additional benefit of solving this problem

By selecting the tweets per hour instead of per day, we were multiplied the number of observations by a factor of 24. This is beneficial for our data analysis.

Thanks for reading. In case we indicate new problems, we will post a new blog post.
