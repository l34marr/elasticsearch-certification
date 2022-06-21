# Overview

[Topics](https://www.elastic.co/training/elastic-certified-engineer-exam) [Kibana Training](https://www.elastic.co/training/kibana-fundamentals) [Beginner Video](http://ela.st/beginners-table-of-contents)

HTTP 和 transport 兩種溝通模式 http.port (9200-9299) transport.tcp.port (9300-9399)

```
POST _reindex
{
  "source": {
    "index": "produce_index"
  },
  "dest": {
    "index": "produce_v2"
  }
}
```

```
PUT product_v2/_mapping
{
  "runtime": {
    "total": {
      "type": "double",
      "script": {
        "source": "emit(doc['unit_price'].value* doc['quanty'].value)"
      }
    }
  }
}

GET product_v2/_search
{
  "size": 0,
  "aggs": {
    "total_expense": {
      "sum": {"field": "total"}
    }
  }
}
```

# JVM

每個發行版會內含一份 OpenJDK (CentOS /usr/share/elasticsearch/jdk)
```
$ /usr/share/elasticsearch/jdk/bin/java --version
openjdk 17.0.2 2022-01-18
OpenJDK Runtime Environment Temurin-17.0.2+8 (build 17.0.2+8)
OpenJDK 64-Bit Server VM Temurin-17.0.2+8 (build 17.0.2+8, mixed mode, sharing)
```

# Inverted Index

```
"match": {"geoip.country_name": "united"}
"sort": {"geoip.country_name": {"order": "asc"}}
```
以上會產生錯誤, 改用 Keyword Field -- "sort": {"geoip.country_name.keyword" ...} 因為內建會產生 Doc Value 用於 aggs 和 sort

沒打算做索引或聚合的欄位, 可以指定取消來加速, 甚至完全失效
```
"http_version": {
  "type": "keyword",
  "index": false,
  "doc_values": false,
  "enabled": false
}
```

# Mapping

把 first_name last_name 執行 copy_to 到 full_name 可以啟動索引, 但不會出現在 `_source`

[null_value](http://elastic.co/guide/en/elasticsearch/reference/current/null-value.html) 可以指定預設值, 但 `[ ]` 並沒包含 Null 所以不會變成預設值

# Relevance

Search Totoal Hits: since 7.0, ES limits the total counts to 10,000.

```
GET _search
{
  "track_total_hits": true
}
```

Default scoring algorithm is [BM25](https://www.elastic.co/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables)

```
GET my-index/_search
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "new music",
        "slop": 1
      }
    }
  }
}
```

the slop parameter tells how far apart terms are allowed to be while still considering the document a match: *new fresh music* is included

```
GET my-index/_search
{
  "query": {
    "multi_match": {
      "query": "new music",
      "fields": [
        "title",
        "content",
        "category"
      ],
      "type": "best_field" /* use "phrase" to improve precision */
    }
  }
}
```

```
GET news_headlines/_search
{
  "query": {
    "multi_match": {
      "query": "party planning",
      "fields": [
        "headline", "short_description"
      ],
      "type": "phrase"
    }
  }
}
```

[Beginner Tutorial](http://ela.st/beginners-table-of-contents): Click on Part 2 ; Video 15:00 - 21:46 ; News Category Dataset from Kaggle news_headlines ; Click on Part 3 ; Video 27:00

```
GET news_headlines/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category", ## RETURN KEYWORD LIST and its count
        "size": 100  ## LIMIT to first 100 terms
      }
    }
  }
}
```

```
GET ecommerce_data/_search
{
  "size": 0,
  "aggs": {
    "transactions_by_8_hrs": {
      "date_histogram": {
        "field": "InvoiceDate",
        "fixed_interval": "8h"  ### "calendar_interval": "day"
      }
    }
  }
}
```

```
GET my-index/_search
{
  "query": {
    "match": {
      "content": {
        "query": "shard",
        "fuzziness": 1 /* can set to "auto" */
      }
    }
  }
}
```

```
GET news_headlines/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_phrase": { "headline": "Michelle Obama" }
        },
        {
          "match": { "category": "POLITICS" } /* "must_not":[{"match":{"category":"WEDDING"}}] */
        } /* "should":[{}] */
      ] /* optional the level of "must" -- "filter":{"range":{"date":{"gte":"","lte":""}}} */
    }
  }
}
```

# Aggregation

* Metrics: sum, avg, stats, cardinality
* Buckets: date_histogram [fix_interval, calendar_interval], histogram, range, terms
* Terms: 


```
GET my-index/_search
{
  "aggs": {
    "transactions_per_custom_price_ranges": {
      "range": {
        "field": "UnitPrice",
        "ranges": [
          {
            "to": 50
          },
          {
            "from": 50,
            "to": 200
          },
          {
            "from": 200
          }
        ]
      }
    }
  }
}
```

```
GET my-index/_search
{
  "aggs": {
    "top_5_customers": {
      "terms": {
        "field": "CustomerID",
        "size": 5 /* optionally to add below -- "order": { "_count": "asc" } */
      }
    }
  }
}
```

```
GET my-index/_search
{
  "aggs": {
    "transactions_per_day": {
      "date_histogram": {
        "field": "InoviceDate",
        "calendar_interval": "day" /* optionally to add below -- "order": { "daily_revenue": "desc" } */
      },
      "aggs": {
        "daily_revenue": {
          "sum": {
            "script": {
              "source": "doc['UnitPrice'].value * doc['Quanity'].value"
            }
          }
        },
        "number_of_unique_customers_per_day": {
          "cardinality": {
            "field": "CustomerID"
          }
        }
      }
    }
  }
}
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

index.max_result_window (defaults to 10,000) can be updated via [API](https://discuss.elastic.co/t/elasticsearch-does-not-take-index-max-result-window-in-elasticsearch-yml/143399), not elasticsearch.yml
search_after or scroll for [more than 10000 results](https://stackoverflow.com/questions/59503012/python-api-for-elastic-search-getting-10000-in-response-every-time)

# Elastic Cloud

[Node Roles](http://ithelp.ithome.com.tw/articles/10240909): Master-Eligible, Data, Ingest, Machine Learning, Transform, Coordinating

```
GET _cluster/health
GET _cluster/health?level=indices
GET _cluster/health?wait_for_status=green
```
```
PUT /my-index
{
  "mappings": {
    "properties": {
      "age":   { "type": "integer" },
      "email": { "type": "keyword" },
      "name":  { "type": "text" }
  }
}
PUT /my-index/_mapping
{
  "properties" : {
    "skill": {
      "type": "keyword",
      "index": false
    }
  }
}
GET /my-index/_mapping
GET /my-index/_mapping/field/skill
```
index.maaping.total_fields.limit
```
POST /my-index/_doc
PUT  /my-index/_doc/1    # updating if existing

POST /my-index/_create/1
PUT  /my-index/_create/1 # document already exists if existing
```
```
$ cat requests
{ "index" : { "_index" : "my-index", "_id" : "1" } }
{ "field1" : "value1" }
$ curl -s -H "Content-Type: application/x-ndjson" -XPOST localhost:9200/_bulk --data-binary "@requests"; echo
```
```
POST /_aliases
{
  "actions": [
    { "remove": { "index": "demo1", "alias": "staying" }},
    { "add":    { "index": "demo2", "alias": "staying" }}
  ]
}
```

# Data Stream

ILM to automate the backing indices management: move older indices to less expensive hardware
* index template: settings for backing indices
* @timestamp field, mapped as date or date_nanos field type
* read / write requests will be handled differently
* rollover creates a new backing index that becomes the stream's new write index
* operation (shrink or restore) can change a backing index's name, which do not remove a backing index from its data stream
* append-only: when needed, use the udpate by query and delete by query APIs

# Search Template

https://www.elastic.co/guide/en/elasticsearch/reference/8.2/search-template.html


# Earthquake

https://github.com/elastic/examples/tree/master/Exploring%20Public%20Datasets/earthquakes
