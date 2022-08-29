Online topic modeling (sometimes called "incremental topic modeling") is the ability to learn incrementally from a mini-batch of instances. Essentially, it is a way to update your topic model with data on which it was not trained on before. In Scikit-Learn, this technique is often modeled through a `.partial_fit` function, which is also used in BERTopic. 

In BERTopic, there are two main goals for using this technique.

1. To reduce the memory necessary for training a topic model. 
2. To continuously update the topic model as new data comes in. 
3. To continuously find new topics as new data comes in. 

In BERTopic, online topic modeling can be a bit tricky as there are several steps involved in which online learning needs to be made available. To recap, BERTopic consists of the following 6 steps:

1. Extract embeddings
2. Reduce dimensionality
3. Cluster reduced embeddings
4. Tokenize topics
5. Extract topic words
6. Diversify topic words

For some steps, an online variant is more important than others. Typically, in step 1 we use pre-trained language models that are in less need for continuous updates. This means that we can use an embedding model like Sentence-Transformers for extracting the embeddings and still use it in an online setting. Similarly, step 5 and 6 do not necessarily need online variants since they are built upon step 4, the tokenization. If that tokenization is by itself incremental, then so will steps 5 and 6. 

This means that we will need online variants for steps 2 through 4. Steps 2 and 3, dimensionality reduction and clustering, can be modeled through the use of Scikit-Learn's `.partial_fit` function. In other words, it supports any algorithm that can be trained using `.partial_fit` since these algorithms can be trained incrementally. For example, incremental dimensionality reduction can be achieved using Scikit-Learn's `IncrementalPCA` and incremental clustering with `MiniBatchKMeans`.

Lastly, we need to develop an online variant for step 5, tokenization. In this step, a Bag-of-words representation is created through the `CountVectorizer`. However, as new data comes in, its vocabulary will need to be updated. For that purpose, `bertopic.vectorizers.OnlineCountVectorizer` was created that not only updates out-of-vocabulary words but also implements decay and cleaning functions to prevent the sparse bag-of-words matrix to become too large in size. Most notably, the `decay` parameter is a value between 0 and 1 to weight the percentage of frequencies that the previous bag-of-words matrix should be reduced to. For example, a value of `.1` will decrease the frequencies in the bag-of-words matrix with 10% at each iteration. This will make sure that recent data has more weight than previously iterations. Similarly, `delete_min_df` will remove certain words from its vocabulary if its frequency is lower than a set value. This ties together with the `decay` parameter as some words will decay over time if not used. 



## **Example**

Online topic modeling in BERTopic is rather straightforward. We first need to have our documents in split in chunks such that we can train and update our topic model incrementally. 

```python
from sklearn.datasets import fetch_20newsgroups

# Prepare documents
all_docs = fetch_20newsgroups(subset=subset,  remove=('headers', 'footers', 'quotes'))["data"]
doc_chunks = [all_docs[i:i+1000] for i in range(0, len(all_docs), 1000)]
```

Here, we created chunks of 1000 documents to be fed in BERTopic. Then, we will need to define a number of sub-models that support online learning. Specifically, we are going to be using `IncrementalPCA`, `MiniBatchKMeans`, and the `OnlineCountVectorizer`:

```python
from sklearn.cluster import MiniBatchKMeans
from sklearn.decomposition import IncrementalPCA
from bertopic.vectorizers import OnlineCountVectorizer

# Prepare sub-models that support online learning
umap_model = IncrementalPCA(n_components=5)
cluster_model = MiniBatchKMeans(n_clusters=50, random_state=0)
vectorizer_model = OnlineCountVectorizer(stop_words="english", decay=.01)
```

!!! tip Tip
    You can use any other dimensionality reduction and clustering algorithm as long as they have a `.partial_fit` function. Moreover, you can use dimensionality reduction algorithms that do not support `.partial_fit` functions but do have a `.fit` function to first train it on a large amount of data and then continously add documents. The dimensionality reduction will not be updated but may be trained sufficiently to properly reduce the embeddings without the need to continuously add documents.

After having defined our sub-models, we can start training our topic model incrementally by looping over our document chunks:

```python
from bertopic import BERTopic

topic_model = BERTopic(umap_model=umap_model,
                       hdbscan_model=cluster_model,
                       vectorizer_model=vectorizer_model)

# Incrementally fit the topic model by training on 1000 documents at a time
for docs in doc_chunks:
    topic_model.partial_fit(docs)
```

And that is it! During each iteration, you can access the predicted topics through the `.topics_` attribute. Do note though that only the most recent batch of documents are tracked. If you want to be using online topic modeling for low-memory use cases, then it is advised to also update the `.topics_` attribute. Otherwise, variations such as hierarchical topic model will not work. 

```python
# Incrementally fit the topic model by training on 1000 documents at a time and track the topics in each iteration
topics = []
for docs in doc_chunks:
    topic_model.partial_fit(docs)
    topics.extend(topic_model.topics_)

topic_model.topics_ = topics
```

!!! note
    Do note that in BERTopic it is not possible to use `.partial_fit` after the `.fit` as they work quite differently with respect to internally updating topics, frequencies, representations, etc. 