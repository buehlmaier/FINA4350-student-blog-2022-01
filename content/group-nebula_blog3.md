---
Title: Dataset for model training (Group Nebula)
Date: 2022-04-22
Category: Progress Report
---

By Group "Nebula"

## Blog 3: Dataset for model training

### Introduction
In this blog post, we will detail our dataset preparation for model training. We want to prepare different datasets using different preprocessing techniques in NLP and compare their performance in machine learning model.

### Data Cleaning
We first clean the text data by removing English stopwords from the NTLK pachage.

```python
from nltk.corpus import stopwords

def remove_stopwords(input_text):
        stopwords_list = stopwords.words('english')
        # Some words which might indicate a certain sentiment are kept via a whitelist
        whitelist = ["n't", "not", "no"]
        words = input_text.split()
        clean_words = [word for word in words if (word not in stopwords_list or word in whitelist) and len(word) > 1] 
        return " ".join(clean_words)
```

### Exploratory Data Analysis (EDA)
We then perform exploratory data analysis (EDA) to get a glimpse of the text data we are working on. We plot the class count, word count and word cloud.

### Tokenization
We tokenize the text data we get from previous steps using the the Keras Package.

```python
from keras.preprocessing.text import Tokenizer

tk = Tokenizer(num_words = 10000,
               filters = '!"#$%&()*+,-./:;<=>?@[\\]^_`{|}~\t\n',
               lower = True,
               split = " ")

tk.fit_on_texts(X1_train)

```

### One-hot encoding
After tokenization, we encode the tokenized words using one-hot encoding.

```python
import numpy as np

def one_hot_seq(seqs, nb_features = 10000):
    ohs = np.zeros((len(seqs), nb_features))
    for i, s in enumerate(seqs):
        ohs[i, s] = 1.
    return ohs
```

### Bag-of-words (Bow)
We also explore Bag-of-Words as an alternative to one-hot encoding. The cleaned text is encoded using Bag-of-words in which the frequency of individual word is recorded.

```python
from sklearn.feature_extraction.text import CountVectorizer

vec = CountVectorizer(stop_words='english', max_features=10000)
X4_train_bow = vec.fit_transform(X4_train)
X4_test_bow = vec.transform(X4_test)
```

### Sentiment Analysis
We perform sentiment analysis on the cleaned text to obtain sentiment scores.

```python
from nltk.sentiment.vader import SentimentIntensityAnalyzer

sid = SentimentIntensityAnalyzer()
def get_sentiment(row, mode):
    sentiment_score = sid.polarity_scores(row)
    positive_meter = sentiment_score['pos']
    negative_meter = sentiment_score['neg']
    neutral_meter = sentiment_score['neu']
    compound_meter = sentiment_score['compound']
    
    if(mode=='positive'):
        return positive_meter
    if(mode=='neutral'):
        return neutral_meter
    if(mode=='negative'):
        return negative_meter
    if(mode=='compound'):
        return compound_meter
    raise ValueError
```

### Datasets for Model Training
The above approaches (i.e. tokenization, one-hot encoding, bad-of-words and sentiment analysis) in data-preprocessing are used to prepare different dataset for model training. The following are the datasets:

- Dataset 1: One-hot Encoding (10000 features)
- Dataset 2: One-hot Encoding (20000 features)
- Dataset 3: One-hot Encoding (50000 features)
- Dataset 4: Bag-of-words (10000 features)
- Dataset 5: Bag-of-words (20000 features)
- Dataset 6: Bag-of-words (50000 features)
- Dataset 7: Sentiment Scores (Positive, Neutral, Negative)
- Dataset 8: Sentiment Scores (Compound)
- Dataset 9: Tokenized (10000 features)

We will test the performance of these datasets in different machine leaning model.