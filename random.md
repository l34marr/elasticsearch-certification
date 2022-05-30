# JVM

每個發行版會內含一份 OpenJDK (CentOS /usr/share/elasticsearch/jdk)
```
$ /usr/share/elasticsearch/jdk/bin/java --version
openjdk 17.0.2 2022-01-18
OpenJDK Runtime Environment Temurin-17.0.2+8 (build 17.0.2+8)
OpenJDK 64-Bit Server VM Temurin-17.0.2+8 (build 17.0.2+8, mixed mode, sharing)
```

# Distributed Scoring

## TF vs IDF
Term Frequency 例如第一篇文件中，被我們篩選出兩個重要名詞，分別為「健康」、「富有」，「健康」在該篇文件中出現 70 次，「富有」出現 30 次，那「健康」的 tf = 70 / (70+30) = 70/100 = 0.7，而「富有」的 tf = 30 / (70+30) = 30/100 = 0.3；在第二篇文件裡，同樣篩選出兩個名詞，分別為「健康」、「富有」，「健康」在該篇文件中出現 40 次，「富有」出現 60 次，那「健康」的 tf = 40 / (40+60) = 40/100 = 0.4，「富有」的 tf = 60 / (40+60) = 60/100 = 0.6，tf 值愈高，其單詞愈重要。
Inverse Document Frequency 換個角度來看，例如有 100 個網頁，「健康」出現在 10 個網頁當中，而「富有」出現在 100 個網頁當中，那麼「健康」的 idf = log ( 100/10 ) = log 100 – log 10 = 2 – 1 = 1，而「富有」的 idf = log (100/100) = log 100 – 1og 100 = 2 – 2 = 0。所以，「健康」出現的機會小，與出現機會很大的「富有」比較起來，便顯得非常重要。
若某個單詞愈是集中出現在某幾份文件中，則 IDF 就愈大，其之於整個語料庫而言就愈重要。反之，當某個單詞在大量文件中都出現，它的 IDF 就愈小，代表這個單詞越是一般。

## Term-Document Matrix
將 tf 和 idf 相乘起來，就可以反映出一個單詞在語料庫中對於一份文件有多麼重要。
```
from preprocessing import preprocess_text

# sample documents
document_1 = "This is the first sentence!"
document_2 = "This is my second sentence."
document_3 = "Is this my third sentence?"

# corpus of documents
corpus = [document_1, document_2, document_3]

# preprocess documents
processed_corpus = [preprocess_text(doc) for doc in corpus]
```
```
from sklearn.feature_extraction.text import TfidfVectorizer

# initialise TfidfVectorizer
vectoriser = TfidfVectorizer(norm = None)

# obtain weights of each term to each document in corpus (ie, tf-idf scores)
tf_idf_scores = vectoriser.fit_transform(processed_corpus)

# get vocabulary of terms
feature_names = vectoriser.get_feature_names()
corpus_index = [n for n in processed_corpus]

import pandas as pd

# create pandas DataFrame with tf-idf scores: Term-Document Matrix
df_tf_idf = pd.DataFrame(tf_idf_scores.T.todense(), index = feature_names, columns = corpus_index)
print(df_tf_idf)
```
[dfs_query_then_fetch](https://www.elastic.co/blog/understanding-query-then-fetch-vs-dfs-query-then-fetch)