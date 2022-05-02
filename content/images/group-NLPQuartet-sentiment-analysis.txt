---
Title: Sentiment Analysis (Group NLPQuartet) 
Date: 2022-05-02 11:42
Category: Progress Report
---

By Group "NLPQuartet", written by Chiu Chun Ho Horace


This blog covers our sentiment analysis process. The main goal of sentiment analysis is to automatically determine whether a text leaves a positive, negative, or neutral impression. At first, we wanted to train a sentiment analysis model by ourselves. However, we had only limited numbers of data, especially for news, so, it was not very feasible to train a model. Therefore, we finally decided to use the following pretrained models: TextBlob, NLTK Vader, Afinn, Flair and FinBERT. 
 *Link to code can be found here:* [NLPQuartet_code](https://github.com/crystal-kwan/FINA4350-NLPQuartet/blob/main/Code.ipynb)

## Limitations

The models are easy to apply in general, and the results are straightforward to interpret after we have standardized all the scores to the range between -1 to 1. The only problem was FinBERT as it requires strong computational power to run (10gb ram above). Since none of us had a computer that was strong enough to run this model on our dataset, we decided not to use FinBERT and focused on the other four models instead. 

## Our Approach
A popular formula to calculate the sentiment score (StSc) is:

StSc = (number of positive words - number of negative words)/ total number of words

Instead of using number of positive/ negative words, we substituted the term with the sum of positive phrases' compound scores and the sum of negative phrases'. 

```python
poslen = 0
neglen = 0 
pos = 0
neg = 0

for i in text:
    if sia.polarity_scores(i)['compound'] > 0:
        pos += sia.polarity_scores(i)['compound']
        poslen += 1
    elif sia.polarity_scores(i)['compound'] < 0:
        neg += sia.polarity_scores(i)['compound']
        neglen += 1
                    
if poslen == 0 and neglen == 0:
    score = 0            
elif neglen == 0:
    score = (pos/poslen)
elif poslen == 0:
    score = (neg/neglen)
else:
    score = (pos+neg)/(pos+abs(neg))
```

## Caveats and Application of Models

For TextBlob, NLTK Vader and Afinn, we applied the model on the lemmatized word list of both News and Tweets. As for Flair, as it is a sentence-based model, we applied it on the sentences instead of tokens for News and Tweets. One concern for using these pretrained models is that they have different libraries, some of them may be more suitable for financial context than the others, so that the score returned on the same News/Tweet could be quite different for different models. As a result, some models had a better explanatory power on stock price than the others. Noting this, we did not take an average sentiment score across the four models but rather ran regression separately. To make our models more content-aware, we also customized the sentiment lexicon to assign new sentiment score to special terms and homonyms that would otherwise be tagged as neutral. 

```python
#Before customizing
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
sia = SentimentIntensityAnalyzer()
print(sia.polarity_scores('📉'))
>>> {'neg': 0.0, 'neu': 1.0, 'pos': 0.0, 'compound': 0.0}
```
```python
#After customizing
new_words = {
'fire': 4.0,
'decreasing': -4.0,
'increasing': 4.0,
'decrease': -4.0,
'increase': 4.0,
'rocket':4.0,
'up':4.0,
'down':-4.0,
'bull':4.0,
'bear':-4.0
}

sia.lexicon.update(new_words)
print(sia.polarity_scores('📉'))
>>> {'neg': 0.8, 'neu': 0.2, 'pos': 0.0, 'compound': -0.6124}

```
## Observations and Comments 

We observed that financial news sentiment is largely very neutral. This can be explained by the fact that the articles' contents are too long, and the issue of objectivity may also suggest why sentiment scores are not substantial. We believe that this can be improved by only extracting sentences of non-neutral sentiment as neutral sentences dilute the final sentiment score. We could also derive the sentiment from article titles only as they might be more concise and to the point. 
