---
layout: post
title: 'Elasticsearch routing, routing shard'
description: 'Elasticsearch routing 에 대해 정리'
date: 2024-03-09
draft: false
author: getsolaris
tags: [Elasticsearch, Logstash, routing, index, shard]
---


## metadata 의 routing 이란 ?

`routing` 은 Elasticsearch 에서 색인된 문서를 검색할 때, 특정한 라우팅 값을 사용하여 검색을 수행하는 방법입니다.

**기본 `routing` 값은 문서의 `_id` 값이며, 이 값은 색인 시에 자동으로 생성됩니다.**

`routing` 을 사용하면 특정한 샤드에만 검색을 수행할 수 있습니다.

이를 통해 실무에서 성능 향상 경험이 있었습니다.


---


## 색인 방법

`routing` 을 사용하는 방법은 다음과 같습니다.

```
PUT test/_doc/1?routing=code20240101
{
  "uid": "test@naver.com"
}
```


## 조회 방법

일반적인 조회 방법은 다음과 같습니다.

`routing` 을 설정하지 않았다면, 엘라스틱 서치는 해당 문서가 어디에 있는지 모르기 때문에 모든 샤드에 대해 조회를 수행합니다.

```
GET test/_search

# response
{
    "took" : 30,
    "timed_out" : false,
    "_shards" : {
        "total" : 18,
        "successful" : 18,
        "skipped" : 0,
        "failed" : 0
    },
    .
    .
}
```

하지만, `routing` 을 사용하여 조회를 하게되면 결과는 달라지게 됩니다.

```
GET test/_search?routing=code20240101

# response
{
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
    },
    .
    .
}
```

문서가 색인된 샤드에 대해서만 조회를 수행하게 되어, 성능이 향상됩니다.

실제로 여러개의 샤드가 있는 경우 특정 code 가 code20240101 인 문서를 조회한다고 가정하면 `routing` 을 사용하지 않았을 때와 사용했을 때의 큰 차이를 확인할 수 있습니다.


[ES 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html)를 확인하면 query 에서 routing 를 이용하여 검색과 여러개의 `routing` 을 이용한 검색에 대한 내용을 확인할 수 있습니다.


## Logstash 를 통해 색인 시에 routing 설정하기

필자는 Logstash 를 통해 Elasticsearch 에 색인을 수행하고 있습니다.

Logstash 의 elasticseach output plugin 을 통해 `routing` 을 설정하는 방법은 다음과 같습니다.


```
elasticsearch {
    routing => "%{[@metadata][_routing_id]}"
}
```


## routing 설정 시 어느 샤드에 배치 되었는지 확인하는 방법

특정 `routing` 값을 가진 문서가 어느 샤드에 배치되었는지 직접 확인하는 방법은 `_search_shards` API 를 이용하여 query parameter 로 `routing` 을 같이 넘기는 방법이 있다.

```
GET test/_search_shards?routing=code20240101

# response
{
  "test" : {
    "shards" : [
      [
        {
          "state" : "STARTED",
          "primary" : true,
          "node" : "node1",
          "index" : "test",
          "shard" : 0,
          "allocation_id" : {
            "id" : "allocation_id"
          }
        }
      ]
    ]
  }
}
```


## reference
- https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/search-shards.html