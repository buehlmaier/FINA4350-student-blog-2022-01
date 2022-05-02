---
Title: Takeaways from preprocessing financial news and tweets! (Group NLPQuartet) 
Date: 2022-05-02 10:35
Category: Progress Report
---

By Group "NLPQuartet," written by Kwan Wing Ching and Kwok Tsun Hei


Following our previous post on data collection, this blog covers the processing stage in which we clean, reshape and preprocess the collected data for sentiment analysis later. This post is useful if you are analyzing text from financial newspapers and social media, as we will dive into all our considerations and issues to be aware of. 

## Overall Objective and Workflow

Using the data collected, we needed to select useful attributes (e.g. number of retweets), filter relevant text data, (e.g. Tweets only in English), and preprocess the text (e.g. removing URLs). Next, we remove stop words, tokenize and lemmatize the textual data.

In the sections below, we will explore some of these steps in detail separately for news and Twitter as they present specific processing requirements. *Link to code can be found here:* [NLPQuartet_code](https://github.com/crystal-kwan/FINA4350-NLPQuartet/blob/main/Code.ipynb)

### News Data

For news data processing, we have several observations. 

* It is typical to find inserted links and advertisement text in news articles. We need to clean them out in the beginning, by using str.replace in Pandas series. Next, we removed the useless symbols before we begin tokenization. We have used re.sub here and we have learnt different types of regular expressions (regex) in this step.

```python
news.Content_Clean = news.Content_Clean.map(lambda x: re.sub('((www\.[^\s]+)|(https?://[^\s]+))',' ', x))
news.Content_Clean = news.Content_Clean.map(lambda x: re.sub('@[^\s]+',' ', x))
news.Content_Clean = news.Content_Clean.map(lambda x: re.sub('[,.\'"!+?\-=)(*><&/;:$%]', ' ', x))
news.Content_Clean = news.Content_Clean.map(lambda x: re.sub('[\s]+', ' ', x))
```

* Stop words removal is a standard procedure in text preprocessing, the list of stop words is therefore often disregarded. However, we noticed that some important words are removed by the `from nltk.corpus import stopwords` package, like 'up' , 'down' ,etc. As our subject of analysis is stock price sensitivity, these words are actually indicative of price movements in the financial context. Besides, removing all the negative contractions can significantly impact the sentiment analysis as it leads to false positives. Therefore, we chose to exclude those useful stop words, and on top of that, append useless terms into the stop word list. We have learnt that it is important to customize our stop word list depending on the context.

```python
import nltk
nltk.download('stopwords')
from nltk.corpus import stopwords

def newsStopword(content):   
    stopWords = stopwords.words('english') 
    stopWords.extend(['reuters', 'january', 'february', 'march', 'april', 'may', 'june', 
                      'july', 'august', 'september', 'october', 'november', 'december', 
                      'london', 'summary', 'new york','bengaluru', 'america'])
    keepWords = ['up','down','against', 'above','below','off','over','further','no','not','only']
    stopWords = list(set(stopWords) - set(keepWords))
    return [[i for i in simple_preprocess(str(doc)) 
             if i not in stopWords and len(i) >=1 ] for doc in content]
```
* Tip: Sometimes it takes hours to lemmatize every single word inside a data frame. With the help of multithreading using the package `concurrent.futures`, we can use more threads to perform concurrency. The function will run faster because we have more concurrent CPU workers at the same time.
```python
#Using concurrency to speed up the lemmatization function
with concurrent.futures.ThreadPoolExecutor(max_workers=30) as executor: 
    df['content_lemmatized'] = list(executor.map(_3_Lemmatization, df['content_tokenized']))
```

### Tweets Data
* We observed a lot of duplicate tweets while checking the dataframe that seem to be advertisement bots so we removed multiple occurences of the same tweet content to remove noise from the data.
```python
df = df.drop_duplicates(subset=['content'], keep='first')
```
* We did similar text preprocessing procedure as above but included removals of all username mentions, hashtags and company tags. 

![Picture showing tweet]({static}/images/group-NLPQuartet-sample_tweet.png)

```python
#Convert to lower case
tweet = tweet.lower()
#Remove www.* or https?://*
tweet = re.sub('((www\.[^\s]+)|(https?://[^\s]+))',' ',tweet)
#Remove @username 
tweet = re.sub('@[^\s]+',' ',tweet)
#Remove additional white spaces
tweet = re.sub('[\s]+', ' ', tweet)
#Remove newlines
tweet = tweet.strip('\n')

#Remove all username mentions, hashtags and ticker tags
tweet = ' '.join(re.sub("(@[A-Za-z0-9]+)|(#[A-Za-z0-9]+)|(\$[A-Za-z0-9]+)", " ", tweet).split())
```
* Emojis, a unique feature of tweets, are converted to text with the `emoji` package.
```python
$pip install emoji --upgrade 
import emoji
#replace emoji with text
tweet = emoji.demojize(tweet)
tweet = tweet.replace(":"," ")
tweet = ' '.join(tweet.split())
tweet = tweet.replace('_'," ")
```


## Final Thoughts

There isn’t a one-size-fit-all playbook when it comes to text preprocessing. Though there are many online examples to adopt directly, we believe that it is more important to know the purpose and logic of the standard text preprocessing procedures and to make it 'your own'. 

Moreover, as we discovered from this project, it is difficult to have the foresight of how clean the text should be and the impact different preprocessed outputs will have on sentiment analysis. For example, Flair model works better with sentences, so we should not tokenize the text into words, and punctuations should be retained. We therefore took an iterative approach and constantly modified the steps aforementioned. Let's take an example to illustrate: Originally we ignored all emojis as 1) they are not tokenized anyhow, and 2) the emojis sentiment score are mostly categorized as neutral even though they are interpreted positively/ negatively; e.g. '📉' should not be neutral in the financial context. We agreed later that they should weigh in sentiment analyzing, hence we converted the emojis to text and revised their sentiment scores accordingly. 
