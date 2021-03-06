---
layout: post
title:  "Bulk Insert/Delete/Update data in ElasticSearch"
date:   2017-01-30 08:00:00 -0700
categories: elasticsearch, bulk, jq
---

ElasticSearch is a great search engine database. However, when you need to perform mass inserts / deletes / updates / upserts, the lack of a native SQL query language makes it much harder to do than with a traditional RDBMS like MySQL.

Let's see how to do a batch insert.

<!--more-->

Let's first bulk index data. According to the documentation [ES 1.x](https://www.elastic.co/guide/en/elasticsearch/guide/1.x/bulk.html) or [ES 2.x](https://www.elastic.co/guide/en/elasticsearch/guide/current/bulk.html)

```bash
curl -XPOST 'localhost:9200/_bulk?pretty' -H 'Content-Type: application/json' -d'
{ "index":  { "_index": "hello", "_type": "doc" }}
{ "title": "Hello world" }
{ "index":  { "_index": "hello", "_type": "doc" }}
{ "title": "Hello world 2" }
'
```

You can find the 2 documents that have been created.

```bash
curl -XGET 'localhost:9200/hello/_search?pretty'
> {
  "took" : 10,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "hello",
        "_type" : "doc",
        "_id" : "AVnnCn_StArLYxHTogPv",
        "_score" : 1.0,
        "_source" : {
          "title" : "Hello world"
        }
      },
      {
        "_index" : "hello",
        "_type" : "doc",
        "_id" : "AVnnCn_StArLYxHTogPw",
        "_score" : 1.0,
        "_source" : {
          "title" : "Hello world 2"
        }
      }
    ]
  }
}
```

The response can be a little bit ugly because of all the metadata. Here comes [jq](https://stedolan.github.io/jq/manual/) to parse the JSON on the command line.

Let's say you just want to get the titles:

```bash
curl -XGET 'localhost:9200/hello/_search' | 
jq '.hits.hits[] | 
{title:._source.title}' -c

> {"title":"Hello world"}
{"title":"Hello world 2"}
```

The -c option inlines the output. Note that it does not return an array but several lines of JSON, which will be handy later.

Now let's say you want to update the titles and replace 'world' by 'Everyone':

```bash
curl -XGET 'localhost:9200/hello/_search' | 
jq '.hits.hits[] | 
{ "index":  {_index, _type, _id }}, ._source' -c | 
sed 's/world/Everyone/g' | 
curl -XPOST http://localhost:9200/_bulk --data-binary @-
```

The 'create' verb is useful if you want to create a document only if it does not exists in the index. For instance, you can easily copy data from one index to another and even from one cluster to another:

```bash
curl -XGET 'localhost:9200/hello/_search' | 
jq '.hits.hits[] | 
{ "create":  {_index:"new", _type, _id }}, ._source' -c | 
curl -XPOST http://localhost:9200/_bulk --data-binary @-
```

or perform a batch delete:

```bash
curl -XGET 'localhost:9200/hello/_search' | 
jq '.hits.hits[] | 
{ "delete":  {_index:"new", _type, _id }}' -c | 
curl -XPOST http://localhost:9200/_bulk --data-binary @-
```

All these operations can be pretty handly if you need to perform not 2 updates but 100 000. I did a test to bulk create 100K documents and it just took a few minutes, depending of course of the speed of your connection if accessing a remote cluster and the characteristics of your cluster.