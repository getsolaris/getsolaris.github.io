---
layout: post
title: 'Elasticsearch Index Rolling, reindex'
description: 'Elasticsearch 색인 시 날짜별로 색인하기 (with Index Template, Reindex)'
date: 2024-03-09
draft: false
author: getsolaris
tags: [Elasticsearch, reindex, index_template]
---


## 소개

하나의 인덱스(테이블)에 수억건의 데이터를 저장하고 있으며, 이를 더 효율적으로 관리하기 위한 전략을 도입하고자 합니다.

이번 글에서는 엘라스틱 서치(Elasticsearch)의 `Index Template` 기능을 활용하여
날짜별로 데이터를 관리하는 `Index Rolling` 방법과 기존 데이터를 재색인(`reindex`) 하는 방법을 기록하고자 합니다.


---


## 목표

테스트용 `posts.v7` 인덱스에 저장된 데이터를 재색인하여 시간대별로 나누는 것입니다.

예를 들어, 2024년 03월 09일 13시 10분의 데이터는 `posts.v8-2024-03-09-13-10` 인덱스에,

20분의 데이터는 `posts.v8-2024-03-09-13-20` 인덱스에 저장합니다.


---


## Index Template 설정

1. Kibana의 Stack Management > Index Management > Templates 페이지로 이동하여 Create template을 클릭합니다.
2. 템플릿 이름을 정하고, 인덱스 이름의 패턴을 입력합니다. `posts.v8-날짜-시간대` 형식으로 인덱스를 생성하기 위해 `posts.v8-*`을 접두사로 설정합니다.

![1](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/main/images/2024-03-09-elasticsearch-index-rolling-reindex/2.png)

3. Primary Shard와 Replica Shard의 수를 지정하고, 필요한 analyzer 설정을 진행합니다.

![2](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/main/images/2024-03-09-elasticsearch-index-rolling-reindex/3.png)

4. 인덱스 맵핑을 설정합니다. GUI를 통해 맵핑 필드를 추가할 수 있으며, 기존 v7 버전의 설정 파일을 `Load JSON` 기능으로 불러올 수도 있습니다.

![3](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/main/images/2024-03-09-elasticsearch-index-rolling-reindex/4.png)

5. 모든 설정을 확인한 후, 인덱스 템플릿을 생성합니다. 생성 후 `posts.v8-template` 템플릿을 확인할 수 있습니다.

![4](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/main/images/2024-03-09-elasticsearch-index-rolling-reindex/5.png)


---


## 테스트

인덱스 템플릿이 정상적으로 동작하는지 테스트합니다.

1. 아래의 예제와 같이 새로운 인덱스에 문서를 추가합니다. 인덱스 템플릿 규칙에 따라 설정과 맵핑이 자동으로 적용됩니다.
```json
PUT posts.v8-2024-03-08/_doc/1
{
  "subject": "2024-03-08 rolling"
}

GET posts.v8-2024-03-08/_mapping
```

2. 다른 날짜의 인덱스에도 동일한 테스트를 진행합니다.
```json
PUT posts.v8-2024-03-09/_doc/1
{
  "subject": "date rolling"
}

GET posts.v8-2024-03-09/_mapping
```

`posts.v8-2024-03-08` 인덱스를 수동으로 생성하고 맵핑을 설정하지 않았는데도 ES 의 인덱스 템플릿 규칙에 의해
위에서 정해둔 설정과 맵핑을 그대로 생성되는 것을 확인할 수 있습니다.


Elasticsearch의 검색 API (`_search`)의 멀티테넌시 검색을 이용하여 여러 인덱스에서 동시에 데이터를 검색해 봅니다.
(이는 `*` (asterisk) 검색 또는 `,` (쉼표) 로 구분하여 여러개의 인덱스를 동시에 검색이 가능합니다.)

아래 쿼리는 subject 필드에 "rolling"이 포함된 모든 문서를 검색합니다.

```json
GET posts.v8-*/_search
{
  "query": {
    "match": {
      "subject": "rolling"
    }
  }
}
```

아래 쿼리는 subject 필드에 "2024-03-08"이 포함된 모든 문서를 검색합니다.

```json
GET posts.v8-2024-03-09,posts.v8-2024-03-08/_search
{
  "query": {
    "match": {
      "subject": "2024-03-08"
    }
  }
}
```

---


## Reindex

이제 기존 `posts.v7` 에 있는 데이터를 새롭게 정의한 인덱스 규칙에 따라 재색인 할 계획입니다.
원래는 게시글 작성일 기준으로 하려고 했으나, `@timestamp` 필드를 기준으로 나누고자 합니다.

2024년 03월 05일 07시 51분대 데이터가 몇 건이 있는지 확인합니다.

```json
GET posts.v7/_count
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2024-03-05T07:51:00",
        "lte": "2024-03-05T07:51:59"
      }
    }
  }
}

# Response
{
  "count" : 2157,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}
```

재색인 API (`_reindex`) 를 이용하여
`subject` 와 `@timestamp` 필드만 새로운 `posts.v8-2023-03-05-07-51` 의 인덱스로 재색인을 해보겠습니다.

`wait_for_completion=false` 옵션을 이용하여 프로세스를 백그라운드에서 실행하도록 설정합니다.

https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html#docs-reindex-filter-source

```json
POST _reindex?wait_for_completion=false
{
  "source": {
    "index": "posts.v7",
    "_source": [
      "subject",
      "@timestamp"
    ],
    "query": {
      "range": {
        "@timestamp": {
          "gte": "2024-03-05T07:51:00",
          "lte": "2024-03-05T07:51:59"
        }
      }
    }
  },
  "dest": {
    "index": "posts.v8-2023-03-05-07-51"
  }
}
```

재색인 작업이 비동기적으로 진행되기 때문에, 작업이 완료되었는지 여부와 진행 상황을 확인하기 위해서는 반환된 `task_id`를 사용합니다.
아래는 작업 상태를 확인하는 방법입니다.

```
GET _tasks/<task_id>
```

응답으로 온 task id 를 검색하여 `completed` 가 `true` 인지, `description` 을 통해
우리가 원하는 대로 reindex 가 진행되고 있는지 확인할 수 있습니다.

```json
{
  "completed" : true,
  "task" : {
    "node" : "dO6MoMGzTPOlZisydnCitA",
    "id" : 43945,
    "type" : "transport",
    "action" : "indices:data/write/reindex",
    "status" : {
      "total" : 2157,
      "updated" : 0,
      "created" : 2157,
      "deleted" : 0,
      "batches" : 3,
      "version_conflicts" : 0,
      "noops" : 0,
      "retries" : {
        "bulk" : 0,
        "search" : 0
      },
      "throttled_millis" : 0,
      "requests_per_second" : -1.0,
      "throttled_until_millis" : 0
    },
    "description" : "reindex from [posts.v7] to [posts.v8-2023-03-05-07-51][_doc]",
    "start_time_in_millis" : 1709967998514,
    "running_time_in_nanos" : 4184516658,
    "cancellable" : true,
    "headers" : { }
  },
.
.
}
```


현재 Elasticsearch에서는 `_reindex` 작업 시 동적으로 대상 인덱스 이름을 지정하는 기능이 제한적입니다.
따라서 특정 패턴에 따라 인덱스 이름을 생성하고자 한다면 별도의 스크립트 작성이 필요하거나, 데이터를 새로 삽입하는 방법을 고려해야 합니다


## reference
- https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html
- https://www.elastic.co/guide/en/elasticsearch/painless/7.17/painless-reindex-context.html 