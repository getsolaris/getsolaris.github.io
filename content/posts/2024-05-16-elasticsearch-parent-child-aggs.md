---
layout: post
title: 'Elasticsearch Parent-child 모델링에서 집계 쿼리 사용하기'
description: 'Elasticsearch Aggregation Query, Parent-child 모델링에서 집계 쿼리 사용하기'
date: 2024-05-17
draft: false
author: getsolaris
tags: [Elasticsearch, parent-child, aggs, aggregations]
---

## Elasticsearch Parent-child 모델링에서 집계 쿼리 사용하기

Elasticsearch 에서는 집계 (Aggregations) 쿼리를 제공합니다. 

이를 이용하여 데이터를 그룹화하거나 통계를 내는 등 다양한 분석을 수행할 수 있습니다. 

Elasticsearch 의 Parent-child 모델링에서 집계 쿼리를 사용하는 방법입니다.


## 집계 쿼리란 ?

> Elasticsearch 의 공식 문서에서는 집계 쿼리를 다음과 같이 정의하고 있습니다. (번역본)

집계는 데이터를 지표, 통계 또는 기타 분석으로 요약합니다. 집계는 다음과 같은 질문에 답하는 데 도움이 됩니다.

- 내 웹사이트의 평균 로드 시간은 얼마나 됩니까?
- 거래량을 기준으로 볼 때 가장 가치 있는 고객은 누구입니까?
- 내 네트워크에서 대용량 파일로 간주되는 것은 무엇입니까?
- 각 제품 카테고리에는 몇 개의 제품이 있나요?

 
Elasticsearch는 집계를 다음 세 가지 범주로 구성합니다.

- 필드 값에서 합계 또는 평균과 같은 지표를 계산하는 지표 집계입니다.
- 필드 값, 범위 또는 기타 기준에 따라 문서를 버킷(빈이라고도 함)으로 그룹화하는 버킷 집계입니다.
- 문서나 필드 대신 다른 집계에서 입력을 받는 파이프라인 집계입니다.


## 시나리오
1. 회원이 있고, 회원은 여러 개의 주문을 가질 수 있습니다.
   2. 회원과 주문은 Parent-child 관계로 구성되어 있습니다.
3. 회원 이메일에 `@naver.com` 이 포함된 회원의 총 주문 금액을 구한다.

더욱 다양하게 집계 쿼리를 사용할 수 있지만 간단하게 위 시나리오를 예로 들어 집계 쿼리를 사용해봅니다.


## 셋팅
- Elasticsearch 7.17.18
  - master x3
- Kibana 7.17.18

## 인덱스 설정
```
PUT users
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "email": {
        "type": "text",
        "analyzer": "standard"
      },
      "order_price": {
        "type": "double"
      },
      "join_fields": {
        "type": "join",
        "relations": {
          "user": "orders"
        }
      }
    }
  }
}
```

이메일을 앞 뒤로 나누어 검색하기 위해 `standard` 분석기를 사용하였습니다.

```
POST _analyze
{
  "analyzer": "standard",
  "text": "test1@naver.com"
}


# response
{
  "tokens" : [
    {
      "token" : "test1",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "naver.com",
      "start_offset" : 6,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 1
    }
  ]
}
```



## bulk 유저 데이터 삽입
```
PUT users/_doc/test1@naver.com?routing=naver.com
{
  "email": "test1@naver.com",
  "join_fields": {
    "name": "user"
  }
}

PUT users/_doc/test2@naver.com?routing=naver.com
{
  "email": "test2@naver.com",
  "join_fields": {
    "name": "user"
  }
}

PUT users/_doc/test3@naver.com?routing=naver.com
{
  "email": "test3@naver.com",
  "join_fields": {
    "name": "user"
  }
}

PUT users/_doc/test11@gmail.com?routing=gmail.com
{
  "email": "test11@gmail.com",
  "join_fields": {
    "name": "user"
  }
}

PUT users/_doc/test22@gmail.com?routing=gmail.com
{
  "email": "test22@gmail.com",
  "join_fields": {
    "name": "user"
  }
}

PUT users/_doc/test111@abcd.com?routing=abcd.com
{
  "email": "test111@abcd.com",
  "join_fields": {
    "name": "user"
  }
}
```

그 후 Elasticsearch 에서 아래와 같이 데이터가 삽입되었는지 확인합니다.

```
GET users/_search
```

총 6개의 데이터가 삽입되었습니다.

이제 주문 데이터를 삽입합니다.

```
POST users/_bulk?routing=naver.com
{"index":{},"routing":"test1@naver.com"}
{"order_price":100,"join_fields":{"name":"orders","parent":"test1@naver.com"}}
{"index":{},"routing":"test1@naver.com"}
{"order_price":1000,"join_fields":{"name":"orders","parent":"test1@naver.com"}}
{"index":{},"routing":"test1@naver.com"}
{"order_price":2510,"join_fields":{"name":"orders","parent":"test1@naver.com"}}
{"index":{},"routing":"test1@naver.com"}
{"order_price":3000,"join_fields":{"name":"orders","parent":"test1@naver.com"}}
{"index":{},"routing":"test2@naver.com"}
{"order_price":150000,"join_fields":{"name":"orders","parent":"test2@naver.com"}}

POST users/_bulk?routing=gmail.com
{"index":{},"routing":"test11@gmail.com"}
{"order_price":50000,"join_fields":{"name":"orders","parent":"test11@gmail.com"}}
{"index":{},"routing":"test22@gmail.com"}
{"order_price":150000,"join_fields":{"name":"orders","parent":"test22@gmail.com"}}

POST users/_bulk?routing=abcd.com
{"index":{},"routing":"test111@abcd.com"}
{"order_price":3500,"join_fields":{"name":"orders","parent":"test111@abcd.com"}}
```

## @naver.com 이메일을 가진 회원의 총 주문 금액 구하기
```
GET users/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "email": "@naver.com"
          }
        }
      ]
    }
  }, 
  "aggs": {
    "orders": {
      "children": {
        "type": "orders"
      },
      "aggs": {
        "total_order_price": {
          "sum": {
            "field": "order_price"
          }
        }
      }
    }
  }
}

# response
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "orders" : {
      "doc_count" : 5,
      "total_order_price" : {
        "value" : 156610.0
      }
    }
  }
}
```

총 6명의 회원이 있지만, `@naver.com` 이메일을 가진 회원은 3명입니다.

## 정리
1. `must` 절에 `match` 쿼리를 사용하여 standard 분석기에 분리된 term 중 `@naver.com` 이 포함된 회원을 검색합니다.
2. `children` 집계를 사용하여 `orders` 타입의 자식 문서를 검색합니다.
3. `sum` 집계를 사용하여 `order_price` 필드의 총 합계를 구합니다.


## Reference
- [Elasticsearch Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)