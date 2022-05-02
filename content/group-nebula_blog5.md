---
Title: Data Re-Cleaning (Group Nebula)
Date: 2022-04-27
Category: Progress Report
---

By Group "Nebula"

## Blog 5: Data Re-Cleaning

### Introduction

In this blog post, we will discuss improvements we made to our process from issues we noticed during the process of building the model.

### FinBERT text cleaning

We were concerned that the _entire_ Item 7 section would contain too much information for the model to be able to generalize a relationship from with just ~250 data points. Upon further inspecting several Item 7s, we noticed that there were a lot of "junk" sentences that likely would not contribute to the stock's performance, such as disclaimer sentences (see below for example). These sentences will generally be of neutral sentiment.

> This section and other parts of this Form 10-K contain forward-looking statements that involve risks and uncertainties. The Company's actual results may differ significantly from the results discussed in the forward-looking statements.

On the other hand, sentences that likely contribute to whether a company performs well will generally be of non-neutral sentiment (ie. _increase_ in sales, _decrease_ in debt). As a result, we decided to explore if pruning neutral sentiment sentences from Form 10-Ks would help the model learn better (since it now only has to "look at" more relevant sentences).

We have opted to use FinBERT, a sentiment analysis model trained specifically on financial reports. Since our text data are of a financial context as well, FinBERT should give relatively accurate sentiment labels.

This is confirmed to be true after some preliminary testing:

- `"In addition to historical information, the following discussion contains forward-looking statements that are subject to risks and uncertainties."` shows as neutral
- `"Revenues were 23.5 billion, a decrease of compared to revenues of 24.3 billion in fiscal with net income of 5.2 billion."` shows as negative

Since FinBERT takes in individual sentences as input, I also used spacy's sentence recgoniser (senter) to split the text file into sentences before feeding them into FinBERT. I first imported the relevant packages and the FinBERT model:

```py
from transformers import BertTokenizer, BertForSequenceClassification
from transformers import pipeline
import spacy

# load stuff needed for FinBERT sentiment & spacy's sentence recognizer!
finbert = BertForSequenceClassification.from_pretrained('yiyanghkust/finbert-tone',num_labels=3)
tokenizer = BertTokenizer.from_pretrained('yiyanghkust/finbert-tone')
BERTnlp = pipeline("sentiment-analysis", model=finbert, tokenizer=tokenizer)
spacynlp = spacy.load("en_core_web_sm", disable=["tok2vec", "tagger", "parser", "attribute_ruler", "lemmatizer", "ner"])
spacynlp.enable_pipe("senter")
```

Since I am only using spacy for sentence recognition, I have disabled all other processes to speed up the process.

I then put the text file into spacy's nlp pipeline for sentence recognition, and then iterated through each sentence to generate a sentiment label through FinBERT. I then ignored all sentences with a neutral label and wrote non-neutral sentences into the cleaned file.

```python
# get text from document
with open("TXN_2022-02-04.txt") as infile:
    txt = infile.read()

doc = spacynlp(txt) # feed text into spacynlp for sentence recognition

for sentence in doc.sents:
    label = BERTnlp(sentence.text)[0] # obtain label (positive / negative / neutral) from FinBERT
    if label['label'] == 'neutral': continue # ignore sentences with neutral sentiment
    else: outfile.write(sentence.text + " ")
```

One major drawback of this method was it was _very_ slow (doing this on ~250 documents took over 4 hours to do), which meant it might not be scalable for datasets with even more documents, even after disabling the unnecessary components of spacy's nlp pipeline.

I have tried to optimise the program further by feeding in a list of all sentences in the document to BERTnlp (ie. `BERTnlp([sentence.text for sentence in doc.sents]`) since BERTnlp() also accepts a list of strings, but this was very memory-intensive and caused problems for my computer.

Another limitation of this approach is that there are potentially sentences of neutral sentiment (ie. those that describe a new product being developed by the company) that would also contribute to the performance of the stock, but would be discarded using the FinBERT approach. As a result, the model would be missing out on information that would actually determine a company's next-day performance.

With more time, a "smarter", less time-consuming approach that also keeps all relevant sentences could be developed and used to trim the textual data instead.

### Re-scraping of text data

We found that our original text extraction step in our text data scraping left a lot of table data that could not be easily identified and removed from the raw text. As a result, we proceed to perform cleaning on the HTML files first as they preserve more information that can be used to recognize contents that are not within the main paragraphs. To achieve this, we used beautiful soup for parsing the whole HTML file, and then we pick out elements with their corresponding tags, for example, `soup.findall("p", {"align":"center"})` to find out all the elements that are centered. Since the only time that these document center their text is when it is either a) a page number or b) a subtitle, they can be safely screened out as they would contain no information that is beneficial to the analysis. With this being done on the HTML level instead of working with the plain text, we scrap out the data with a lot more ease.

Although it is not without its fair share of issue as well, occasionally, we would run into problems of HTML encoding and/or parser issue that require us to change the parser used. For example, there were one instance where "lxml" just wouldn't read the HTML properly (possibly because of missing end tags), and we have to change to "html.parser" instead. Not to mention that since sometime the encoding are unique and python's file io are not able to automatically detect the encoding of the file, we have to terminate the program to change the setting for the encoding used. Eventually, we were able to produce higher quality results in terms of removing fragments, but trial-and-error and manual effort were required to adjust settings.
