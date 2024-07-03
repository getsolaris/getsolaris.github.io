---
layout: post
title: 'Elasticsearch Ingest Pipeline 을 사용하는 두가지 방법'
description: 'Elasticsearch Ingest Pipeline 을 사용하여 데이터를 전처리 하는 두가지 방법'
date: 2024-07-03
draft: false
author: getsolaris
tags: [Elasticsearch, ingest, pipeline]
---

Elasitcsearch 에서는 Ingest pipeline 을 사용하여 데이터를 전처리할 수 있습니다.


Ingest pipeline 은 데이터를 전처리하는 방법으로, 데이터를 색인하기 전에 데이터를 변환하거나 필터링할 수 있습니다.


예시로 데이터를 색인할 때 timestamp 를 생성하여 색인하고 싶다면 Ingest pipeline 을 사용하여 timestamp 를 생성할 수 있습니다.

여기서 Ingest pipeline 의 프로세서 중 하나인 `set` 프로세서를 사용하여 timestamp 를 생성하는 방법을 알아보겠습니다.


## Ingest pipeline 생성

Ingest pipeline 을 생성하는 방법은 다음과 같습니다.

```
PUT _ingest/pipeline/create-timestamp-pipeline
{
  "description" : "timestamp 생성",
  "processors" : [
    {
      "set" : {
        "field": "timestamp",
        "value": "{{_ingest.timestamp}}"
      }
    }
  ]
}
```

Ingest pipeline 은 Kibana 의 Stack Management > Ingest Pipelines 에서 생성 및 관리가 가능합니다.

생성을 했다면 이제 데이터를 색인할 때 Ingest pipeline 을 사용하여 데이터를 전처리할 수 있습니다.


## 첫번째 방법 - mapping 에 pipeline 설정

인덱스를 동적 매핑으로 생성할 때, Ingest pipeline 을 설정할 수 있습니다.

[index.default_pipeline](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#index-default-pipeline) 을 사용하여 인덱스 생성 시 Ingest pipeline 을 설정할 수 있습니다.

또한 _settings API 를 사용하여 기본 파이프라인을 설정할 수 있습니다.


```
PUT users
{
  "settings": {
    "default_pipeline": "create-timestamp-pipeline"
  },
  "mappings": {
    "properties": {
      "email": {
        "type": "keyword"
      }
    }
  }
}
```

### 데이터 색인 및 확인

```
# 데이터 색인
POST users/_doc
{
  "email" : "test@naver.com"
}

# 데이터 조회
GET users/_search
{
  "_index" : "users",
  "_type" : "_doc",
  "_id" : "VTENeZAB03TCp4qHlJ6o",
  "_score" : 1.0,
  "_source" : {
    "email" : "test@naver.com",
    "timestamp" : "2024-07-03T14:44:11.542360471Z"
  }
}
```

위와 같이 timestamp 가 정상적으로 들어간 것을 확인할 수 있습니다.


## 두번째 방법 - 색인 시 Ingest pipeline 옵션을 이용

새롭게 users2 인덱스를 생성하고, 색인 시 Ingest pipeline 을 설정하는 방법입니다.

```
PUT users2
{
  "mappings": {
    "properties": {
      "email": {
        "type": "keyword"
      }
    }
  }
}
```

pipeline 은 위에서 생성했던 `create-timestamp-pipeline` 을 사용합니다.

```
# 데이터 색인
POST users2/_doc?pipeline=create-timestamp-pipeline
{
  "email" : "test2@naver.com"
}

# 데이터 조회
GET users2/_search
{
  "_index" : "users2",
  "_type" : "_doc",
  "_id" : "pDEReZAB03TCp4qHCJ6T",
  "_score" : 1.0,
  "_source" : {
    "email" : "test2@naver.com",
    "timestamp" : "2024-07-03T14:47:57.842137840Z"
  }
}
```

위와 같이 맵핑 시 default_pipeline 을 설정하는 방법과 색인 시 pipeline 을 설정하는 방법 두가지 방법으로 Ingest pipeline 을 사용할 수 있습니다.


일반적인 색인 뿐만이 아니라 `_update_by_query` 와 `_reindex` API 를 사용하여 기존 데이터를 업데이트할 때도 Ingest pipeline 을 사용할 수 있습니다.


## 나아가서

[Ingest pipeline processors](https://www.elastic.co/guide/en/elasticsearch/reference/current/processors.html) 에서 다양한 프로세서를 확인할 수 있습니다.


예를 들어 아래와 같이 사용도 가능합니다.
- 여러 데이터 치환
- 텍스트 임베딩
- 등 ..


## 정리
1. Ingest pipeline 을 사용하여 데이터를 전처리할 수 있습니다.
2. Ingest pipeline 을 생성하고, 인덱스 생성 시 default_pipeline 을 설정하거나 색인 시 pipeline 을 설정하여 사용할 수 있습니다.
3. 문서의 업데이트와 재색인 시에도 Ingest pipeline 을 사용할 수 있습니다.
4. 다양한 프로세서를 사용하여 데이터를 전처리할 수 있습니다.


## Reference
- [Elasticsearch Ingest pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html)
- [Elasticsearch Index Modules - Dynamic Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#dynamic-index-settings)
- [Elasticsearch Ingest pipeline processors](https://www.elastic.co/guide/en/elasticsearch/reference/current/processors.html)
- [Elasticsearch Put Pipeline API](https://www.elastic.co/guide/en/elasticsearch/reference/current/put-pipeline-api.html)