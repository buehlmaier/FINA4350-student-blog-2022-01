---
Title: Preparing for news classification training (Group Stonks)
Date: 2022-04-19 01:12
Category: Progress Report
---

By JIANG Zeyu and HUANG Yining of group Stonks

# Introduction

We now have tokenized news data and stock prices data retrieved from yahoo. We want to find out the relations between them using deep learning techniques to sort of predict the stock movement based on news. We have ordered the news by date, and give each date a label which indicates stock movement, i.e. increase, decrease, in a future period from that date.

# Workflow

## 1. Generate feature matrix

Since computer cannot understand natural language directly, we have to turn them into pure numbers. We could do this by constructing a word database that give each word an index, and simply use 0,1 to mark the stock increase and decrease. However, the length of news would not be the same in most cases, we have to decide a max_length for one news piece and add padding to news that is shorter than max_length.

Also, with all the news pieces we gathered, we need to divide them into two sets, i.e. train set and test test, so that we could validate accuracy of our trained model. It should be notice that, feature matrix of train set should share a same words index database with feature matrix of test set, or the mapping will be destroyed.

We first define a class for feature matrix of train set which could accept sentences and labels. It should be able to generate a dictionary named ***wordToIndex*** and its feature matrix.

```python
class FeatureMatrixTrain:
    def __init__(self) -> None:
        # Initial raw matrix
        self.wordToIndex = dict()
        self.indexToWord = []
        self.currentIndex = 0
        self.wordCountByIndex = {}
        self.sentences = []
        self.labels = []
        self.wordToIndexReduced = dict()
        # Final matrix
        self.features = []
        self.fm = None

    def appendToken(self, word: str) -> None:
        # Add word if not exist, or just increase its count
        if word in self.wordToIndex:
            self.wordCountByIndex[self.wordToIndex[word]] += 1
        else:
            # Add word to wordToIndex mapping
            self.wordToIndex[word] = self.currentIndex
            self.indexToWord.append(word)
            # Move pointer
            self.wordCountByIndex[self.currentIndex] = 1
            self.currentIndex += 1

    def appendTokens(self, tokens: List[str], priceChange: float) -> None:
        sentencesIdx = []
        for token in tokens:
            self.appendToken(token)
            sentencesIdx.append(self.wordToIndex[token])
        self.sentences.append(sentencesIdx)
        self.labels.append(priceChange)
```

While iterate through news, we will add news to train set and also test set. To avoid repeatedly iterating, we firstly just add news to test set, and map the words after the feature matrix and wordToIndex are constructed in the train set.

```python
class FeatureMatrixTest:
    def __init__(self) -> None:
        self.sentences = []
        self.labels = []
        self.wordToIndex = dict()
        # Final matrix
        self.features = []
        self.fm = None

    def appendTokens(self, tokens: List[str], priceChange: float) -> None:
        self.sentences.append(tokens)
        self.labels.append(priceChange)

    def mappingWordIndex(self, wordToIndex):
        self.wordToIndex = wordToIndex
        unknown = self.wordToIndex["UNKNOWN"]
        for index, sentence in enumerate(self.sentences):
            newSentence = [
                self.wordToIndex[token] if token in wordToIndex else unknown for token in sentence]
            # Add padding to make equalized sentence, here choose 30 as maximum words in one sentence
            if len(newSentence) > 30:
                newSentence = newSentence[:30]
            else:
                newSentence = newSentence + [len(wordToIndex) - 1] * (30 - len(newSentence))
            # Add label
            newSentence.append(self.labels[index])
            self.features.append(newSentence)
            sys.stdout.write("\rFinished sentence #{}".format(index))
        self.fm = np.matrix(self.features)
        print("\nFinished mapping")
```

It should be noticed that we haven't implement *padding* in feature matirx of train set, since we would do that when implementing reduce vocabulary functions. Just after we've defined these two class, we could iterate our data and add them into two matrices.

```python
for ticker in tickers:
    # News data are in json format
    with open("./data/news/"+ticker+".json", "r") as f:
        news = json.load(f)
    with open("./data/prices_processed/"+ticker+".json", "r") as f:
        prices = json.load(f)
    # Iterate news to match with prices data
    for item in news:
        dateOfNews = datetime.strptime(
            item["updated_at"], "%Y-%m-%dT%H:%M:%SZ")
        dateStrOfNews = dateOfNews.strftime("%Y-%m-%d")
        # Check in prices data
        if dateStrOfNews not in prices:
            continue
        # Divide into two sets
        sampleId += 1
        # Tokenize
        rawContent = item["title"] + " " + item["description"]
        # Remove url first
        rawContent = re.sub(
            r'(https|http)?:\/\/(\w|\.|\/|\?|\=|\&|\%)*\b', '', rawContent)
        # We tokenize and clean our news content here
        tokens = tokenizeContent(rawContent, stopwords)
        if sampleId % 10 == 1:
            # Test set
            fmTest.appendTokens(tokens, prices[dateStrOfNews]['week'])
        else:
            # Train set
            fmTrain.appendTokens(tokens, prices[dateStrOfNews]['week'])
        sys.stdout.write("\rFinished sampling #{}".format(sampleId))
# Construct mapping for test set
fmTest.mappingWordIndex(fmTrain.wordToIndexReduced)
```
```

## 2. Vocabulary reduction

However, there are still some problems like the super large vocabularies would let our deep learning model have too much vectors to follow, resulting in loss of accuracy. Therefore, we decide to reduce our vocabulary. It should include process that sort words index according to words frequency, constructing new index of words, and do the mapping from old index to new index.

Since we haven't done padding process for our train set's feature matrix, the number of columns are matched and we haven't add labels to them, we will do it during our vocabulary reduction process.

Add following functions into class FeatureMatrixTrain. Here we limit the max length of news being 30 words.

```python
def reduceVocab(self, vocabSize: int) -> None:
    # Initialize a smaller wordToIndex
    wordToIndexReduced = {}
    newIndex = 0
    indexMap = {}
    # Sorted and reduce size
    sortedWordCountByIndex = sorted(
        self.wordCountByIndex.items(), key=operator.itemgetter(1), reverse=True)
    sortedWordCountByIndex = sortedWordCountByIndex[:vocabSize]
    print("Sorted by word count")
    # Construct mapping from old index to new index
    for index, count in sortedWordCountByIndex:
        word = self.indexToWord[index]
        wordToIndexReduced[word] = newIndex
        indexMap[index] = newIndex
        newIndex += 1
    print("Constructed mapping from old index to new index for train set")
    # Using capital unknown to avoid duplication
    wordToIndexReduced['UNKNOWN'] = newIndex
    unknown = newIndex
    self.wordToIndexReduced = wordToIndexReduced
    # Remap word in sentences with new index
    for index, sentence in enumerate(self.sentences):
        newSentence = [
            indexMap[index] if index in indexMap else unknown for index in sentence]
        # Add padding to make equalized sentence, here choose 30 as maximum words in one sentence
        if len(newSentence) > 30:
            newSentence = newSentence[:30]
        else:
            newSentence = newSentence + [len(wordToIndexReduced) - 1] * (30 - len(newSentence))
        # Add label
        newSentence.append(self.labels[index])
        self.features.append(newSentence)
        sys.stdout.write("\rFinished sentence #{}".format(index))
    self.fm = np.matrix(self.features)
    print("\nFinished mapping")
```

We won't need to modify any functions in FeatureMatrixTest, we just call fmTest.mappingWordIndex after we call reduceVocab of FeatureMatrixTrain, and pass the reduced wordToIndex.

Then, we could just directly save two featureMatrices and wordToIndex to files for our later training and testing process.

```python
# Save files
with open("./data/wordToIndex", "w") as f:
    json.dump(fmTrain.wordToIndexReduced, f)
with open("./data/train/featureMatrix", "w") as f:
    np.savetxt(f, fmTrain.fm, fmt="%s")
with open("./data/test/featureMatrix", "w") as f:
    np.savetxt(f, fmTest.fm, fmt="%s")
```

## 3. Model Construction

### Choose of neural network

We firstly encounter with the choice between convolutional neural network, CNN and recurrent neural network, RNN, which are classified by characteristics of neural and are kind of evolution of DNN.

DNN is artificial neural network (ANN) with multiple layers between the input and output layers, while CNN is encoded with space dimension, and RNN is encoded with time dimension.

To clarify, we could construct the simplest DNN by connecting each layer fully, that all outputs of previous layer serves as input of next layer, which will not suit our case, since not every word appears each time and last news may not have link with next layer.

When take a look at RNN, it requires there are time-linked relationships with in one entry. For example, if one word is "I" the next word might be "like". However, after our processing, this link has been weakened. For each time step, they may not share same arguments. Therefore, CNN which make the words like a large word cloud and to find relationships would be our choice.

### Construction of CNN model

We start a scratch from pytorch nn module

```python
import torch.nn as nn

class CNN_News(nn.Module):
    def __init__(self, args):
        # Call parent constructor
        super(CNN_News, self).__init__()
        
        # Multiple List of CONV => RELU => POOL layers
        self.convs = nn.ModuleList([nn.Conv2d(1, KERNEL_NUM, (K, EMBEDED_DIMENSION) for K in KERNEL_SIZES)]
        # Dropout and other improvement
        self.dropout = nn.Dropout(dropout)
        self.fc = nn.Linear(len(kernel_sizes) * kernel_dim, output_size)
```

Here, we ignore some settings of parameters, which really depends on your data.

- KERNEL_NUM: You could view this as the number of filters, which we would use filters to shade our data, to focus on a specific area of data. After computing them, we could move to next area.
- EMBEDED_DIMENSION: This parameter is hugely depended on the size of your data, since it would use to assign vectors. Let's put it simply here, it's a number to specify how large would you make your network. If you want to learn how to set this number, you could refer to [AutoEmb: Automated Embedding Dimensionality Search in Streaming Recommendations].

Also, the convs could be illustrate as a for-loop. (Kernel sizes of (2,3,4))

```
self.conv13 = nn.Conv2d(Ci, Co, (2, D))
self.conv14 = nn.Conv2d(Ci, Co, (3, D))
self.conv15 = nn.Conv2d(Ci, Co, (4, D))
```

Then we need to add functions to support max-pooling layer and softmax layer.

Max pooling layer will take maximum value of result (multiple one-dimension vector) of the convolution layer and make them together. Softmax layer will turn the result of pooling layer into classes we want to classify.

To view more details, you could refer to this [PyTorch: Training your first Convolutional Neural Network (CNN) - PyImageSearch](https://pyimagesearch.com/2021/07/19/pytorch-training-your-first-convolutional-neural-network-cnn/).

# What's Next

Now, we've prepared enough for our training process. We could use the resouces to start our training quite easy while we may face a truly low accuracy. We may discover how to adjust parameters for our model and how to correctly use and test our model in our next blog post.