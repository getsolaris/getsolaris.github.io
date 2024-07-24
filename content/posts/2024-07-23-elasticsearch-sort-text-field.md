---
layout: post
title: 'Elasticsearch 문자 정렬'
description: 'Elasticsearch date 타입 필드가 아닌 문자 (text/keyword) 타입의 필드를 정렬하는 방법'
date: 2024-07-24
draft: false
author: getsolaris
tags: [ Elasticsearch, sort ]
---

사이드 프로젝트를 진행하며, 기존 날짜 타입 필드가 아닌 문자 필드를 정렬 할 일이 생겼습니다.

예를 들어 Y-m-d 형식의 date 타입인 필드를 정렬하고자 합니다.

```json
{
  "mapping": {
    "properties": {
      "date": {
        "type": "date"
      }
    }
  }
}
```

그렇다면 아래의 쿼리로 쉽게 정렬할 수 있습니다.

```json
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "date": {
        "order": "asc"
      }
    }
  ]
}
```

하지만, 문자 타입의 필드를 정렬하고자 한다면..

가상으로 `nameEN` 의 필드를 가진 인덱스를 생성하고, `nameEN` 필드를 정렬하고자 합니다.

```json
{
  "mapping": {
    "properties": {
      "nameEN": {
        "type": "text"
      }
    }
  }
}
```

우리는 `nameEN` 의 첫 글자를 기준으로 정렬하고자 합니다.
A-Z 순서로 정렬하고자 한다면, 아래와 같이 쿼리를 작성할 수 있습니다.

```json
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "nameEN": {
        "order": "asc"
      }
    }
  ]
}
```

하지만, ES 에서는 오류를 반환합니다.

```json
{
  "type": "illegal_argument_exception",
  "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [nameEN] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."
}
```

그 이유는, text 타입의 필드는 기본적으로 `fielddata` 옵션이 비활성화 되어 있기 때문입니다.

> `fielddata` 란 역색인된 인덱스를 메모리에 로드하여 검색을 빠르게 하는 기능입니다.

`fielddata` 를 활성화 하면, 메모리를 많이 사용하게 되므로, text 타입의 필드를 정렬할 때는 다른 방법을 사용해야 합니다.

그 방법은 간단하게 해결 가능합니다.

바로 `keyword` 타입으로 맵핑하는 것입니다.

```json
{
  "mapping": {
    "properties": {
      "nameEN": {
        "type": "keyword"
      }
    }
  }
}
```

`keyword` 타입은 `fielddata` 가 아닌 `doc_values` 를 사용합니다.

> `doc_values` 는 역색인된 인덱스를 디스크에 저장하는 방식입니다.

그리고, 쿼리를 아래와 같이 작성하면 정상적으로 정렬이 됩니다.

```json
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "nameEN": {
        "order": "asc"
      }
    }
  ]
}
```

하지만 우리가 원하는 검색을 하기 위해서는 `keyword` 로만은 부족합니다.

여러 analyzer 도 붙여 검색을 하고자 한다면, `text` 타입으로 맵핑해야 합니다.

그럼.. `text` 타입과 `keyword` 타입을 동시에 사용하는 방법은 없을까 ..

## 다중 필드 (multi fields)

바로 다중 필드 (multi fields) 를 사용하여 맵핑을 설정하면 됩니다.

```json
{
  "mapping": {
    "properties": {
      "nameEN": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

아래와 같이 다중 필드를 이용해 여러개의 analyzer 를 사용할 수도 있습니다.

```json
{
  "mapping": {
    "properties": {
      "nameEN": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          },
          "ngram": {
            "type": "text",
            "analyzer": "ngram_analyzer"
          }
        }
      }
    }
  }
}
```

이렇게 검색은 `text` 타입으로 하고, 정렬은 `keyword` 타입으로 할 수 있습니다.

```json
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "nameEN.keyword": {
        "order": "asc"
      }
    }
  ]
}
```

## Ref

- [Elasticsearch Sorting](https://www.elastic.co/guide/en/elasticsearch/reference/current/sort-search-results.html)
- [Elasticsearch doc_values](https://www.elastic.co/guide/en/elasticsearch/reference/current/doc-values.html)
- [Elasticsearch fielddata](https://www.elastic.co/guide/en/elasticsearch/reference/current/fielddata.html)
- [Elasticsearch Multi Fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/multi-fields.html)

