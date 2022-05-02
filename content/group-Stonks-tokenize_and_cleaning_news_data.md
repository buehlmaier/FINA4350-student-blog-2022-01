---
Title: Tokenize and cleaning news data (Group Stonks)
Date: 2022-04-12 11:12
Category: Progress Report
---

By JIANG Zeyu and HUANG Yining of group Stonks

## Background

Since machine cannot understand natural languages directly, a simple and general way to do that it to split sentence into words, and use index to substitute the words. In this way, we could generate a feature matrix which could be used in our deep learning pipeline. However, there would be words that are not relavent and would affect the accuracy of result. There would also be words that represent same idea but of different forms. We have to deal with these problems.

Here, we want to present a simplified but standard pipeline to preprocess and tokenize any data, not just news.

## Pipline

### 1. Tokenize (Word Fragmentation)

Tokenize might be the most simple step in the whole pipeline. It will only split one sentence into several words. For example:

```
Input: "I am a student in HKU"
Output: ["I", "am", "a", "student", "in", "HKU"]
```

It seems simple, but it could be quite complex considering following example:

```
Input: "Machine learning time-consuming"
Output?: ["Machine", "learning", "is", "time-consuming"]
```

You, as a human being, could clearly identify that ***Machine learning*** might be resolved as one token rather than two, or it may not match with ***Machine-learning*** in other sentences. However, after checked a few libraries, none of them have resolved this issue, therefore we have to put the pressure of identifying these terms to machine learning.

```python
import nltk

tokens = nltk.word_tokenize("Machine learning is time-consuming")

>> ["Machine", "learning", "is", "time-consuming"]
```

However, these tokenizer would also make links or emojis into separated tokens, which is definitly what we want.

```python
import nltk

tweet = "RT @angelababy: love you baby! :D http://ah.love #168cm"
print(nltk.word_tokenize(tweet))

>> ['RT', '@', 'angelababy', ':', 'love', 'you', 'baby', '!', ':', 'D', 'http', ':', '//ah.love', '#', '168cm']
```

To resolve the issue, we define some regex expressions to deal with these special cases.

```python
import re

emojis = r"""
    (?:
        [:=;]
        [oO\-]?
        [D\)\]\(\]/\\OpP]
    )"""
regex = [
    emojis,
    r'<[^>]+>', # HTML Tags
    r'(?:@[\w_]+)', # Mention
    r"(?:\#+[\w_]+[\w\'_\-]*[\w_]+)", # Hashtags
    r'http[s]?://(?:[a-z]|[0-9]|[$-_@.&amp;+]|[!*\(\),]|(?:%[0-9a-f][0-9a-f]))+', # URLs
    r'(?:(?:\d+,?)+(?:\.?\d+)?)', # Number
    r"(?:[a-z][a-z'\-_]+[a-z])", # -/`
    r'(?:[\w_]+)',
    r'(?:\S)'
]
```

Also, we need to ignore cases when matching these regex expressions, after matching these expressions we could split them into tokens to satisfy our need.

```python
tokens_re = re.compile(r'(' + '|'.join(regex) + ')', re.VERBOSE | re.IGNORECASE)
emoticons_re = re.compile(r'^' + emojis + '$', re.VERBOSE | re.IGNORECASE)

def tokenize(s):
    return tokens_re.findall(s)

def preprocess(s, lowercase=False):
    tokens = tokenize(s)
    if lowercase:
        tokens = [token if emoticons_re.search(token) else token.lower() for token in tokens]
    return tokens
tweet = "RT @angelababy: love you baby! :D http://ah.love #168cm"
print(preprocess(tweet))

>> ['RT', '@angelababy', ':', 'love', 'you', 'baby', '!', ':D', 'http://ah.love', '#168cm']
```

## 2. Stemming and Lemmatisation

After have every words in one sentence, we need to make sure words of different forms could be mapped into only one word. To simplify, ***play basketball*** and ***playing basketball*** should be of nearly same thing for us, but for computer, they are exactly two things.

In the industry, we have stemming and lemmatisation. Their differences and similarities could be concluded in mainly four aspects.

1. They have same objectives that making word in different forms into stem, basic form of the word, but one by reducing, the other one by transforming.
2. Lemmatisation would be more difficult than stemming, considering we also have to label the part of speech for each word before transforming.
3. Stemming may not always result in a valid word, it may only give a stem. (Just part of a valid word)
4. Stemming may be used in searching more frequently, while **NLP and Text-mining** would be the main focus for lemmatisation.

To do lemmatisation, we could directly use functions provided by nltk packages.

```python
import nltk
from nltk.stem import WordNetLemmatizer

lemmatizer = WordNetLemmatizer()
print(lemmatizer.lemmatize("machines"))
print(lemmatizer.lemmatize("learning"))

>> machine
>> learning
```

We find that ***learning*** isn't been transformed, but in some cases, it should be transformed into ***learn***, which indicates the importence to specify the part of speech. We could do this by using pos integrated in nltk.

```python
import nltk

sentence = "Apple Inc has started internal testing of several Mac models with next-generation M2 chips"
tokens = nltk.word_tokenize(sentence)
tagged_sent = nltk.pos_tag(tokens)
print(tagged_sent)

>> [('Apple', 'NNP'), ('Inc', 'NNP'), ('has', 'VBZ'), ('started', 'VBN'), ('internal', 'JJ'), ('testing', 'NN'), ('of', 'IN'), ('several', 'JJ'), ('Mac', 'NNP'), ('models', 'NNS'), ('with', 'IN'), ('next-generation', 'JJ'), ('M2', 'NNP'), ('chips', 'NNS')]
```

The tags could be referenced [here](https://www.nltk.org/book/ch05.html), and with tags attached to each token, we could pass tags while lemmatizing token, and to unify, we also convert all characters into lower cases.

```python
import nltk
from nltk.stem import WordNetLemmatizer

tokens = nltk.word_tokenize(rawContent)
taggedTokens = nltk.pos_tag(tokens)
processedTokens = []
wnl = WordNetLemmatizer()
for token, tag in taggedTokens:
    # Unifiy words
    wordPos = getWordPos(tag)
    word = wnl.lemmatize(token.lower(), wordPos)
```

## 3. Stopwords

When cleaning news data, using stopwords is another important step, and could significantly increase our accuracy in machine learning process. We could first take a look at stopwords nltk package provide to us, just execute the following commands in your python shell.

```python
import nltk
nltk.download("stopwords")
```

This is a quite simple set which definitely won't match with our use case. Basing on our data, we first calculate the words frequency and pick words that are irrelavent mannually. We would also use some techniques to reduce vocabulary size basing on word frequency.

After these pre-processing steps, you could save these data into any format that is convenient for you to read later on.

# Conclusion

Here we provide a simplifed pipeline to tokenize and clean news data. It's necessary to modify some steps or add other steps based on specific scenario. Since we now have turn one sentence into tokens, we could convert these sentences into feature matrix and use for our machine learning process.