---
layout: post
title: 'Elasticsearch Parent-child 모델링에서 Index Rolling 테스트'
description: 'Elasticsearch Parent-child, Index Rolling 적용 테스트'
date: 2024-04-04
draft: false
author: getsolaris
tags: [Elasticsearch, modeling, parent-child, index-rolling]
---

## 들어가기 전에
[Parent-child](https://www.elastic.co/guide/en/elasticsearch/reference/current/parent-join.html) 모델링은 부모 문서와 자식 문서를 관계를 맺어서 저장하는 방식입니다.

`Parent-child 모델링에서 Index Rolling 적용이 가능할까 ?` 라는 생각을 문득 하게 되어 실험을 해보려고 합니다.

다른 비정규화 방식이면 가능할 것 같지만, Parent-child 모델링은 부모 문서와 자식 문서를 관계를 맺어서 저장하는 방식이기 때문에 Index Rolling 적용이 가능할지 궁금합니다.

그 이유는 하나의 인덱스에 문서의 수가 많아지면, 인덱스의 크기가 커지게 되고, 검색 성능이 저하될 수 있어 Index Rolling 적용이 가능한지 테스트를 해봅니다.


## 셋팅
- Elasticsearch 7.17.18
  - master x3
- Kibana 7.17.18


## 시나리오
1. 회원은 202403 인덱스에 저장한다. (2024년 03월에 가입 했다고 가정)
2. 회원 접속 정보는 202404 인덱스에 저장한다. (2024년 04월에 접속 했다고 가정)
3. 202403 인덱스와 202404 인덱스를 멀티테넌시 검색으로 조회가 되는지 확인한다.


## 실험
1. Parent-child 모델링을 적용한 인덱스를 생성합니다.
여기서는 회원과 회원 접속 정보를 Parent-child 모델링으로 구성합니다.

```
PUT member-202403
{
  "settings": {
    "refresh_interval": "5s", 
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "site_code": {
        "type": "keyword"
      },
      "member_code": {
        "type": "keyword"
      },
      "connected_at": {
        "type": "date"
      },
      "join_fields": {
        "type": "join",
        "relations": {
          "member": "connections"
        }
      }
    }
  }
}
```

- `number_of_shards` 는 현재 노드 수인 3개로 설정하였습니다.
- `number_of_replicas` 는 1로 설정하였습니다.

그렇다면 샤드는 총 6개가 생성됩니다. (3개의 primary shard, 3개의 replica shard)


2. 회원 정보를 색인합니다.
```
PUT member-202403/_doc/member_aa?routing=site_a
{
  "site_code": "site_a",
  "member_code": "member_aa",
  "join_fields": "member"
}
```


3. 회원 접속 정보를 색인합니다.
```
PUT member-202404/_doc/member_aa-2024-04-04?routing=site_a
{
  "site_code": "site_a",
  "member_code": "member_aa",
  "connected_at": "2024-04-04",
   "join_fields": {
    "name": "connections",
    "parent": "member_aa"
  }
}
```

4. 조회 테스트
```
GET member-2024*/_search?routing=site_a
{
  "query": {
    "has_child": {
      "type": "connections",
      "query": {
        "match_all": {}
      }
    }
  }
}

{
  "took" : 7,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```

멀티테넌시와 `routing_id` 를 이용하여 인덱스를 조회하였습니다.
인덱스가 다르면 `routing_id` 를 지정해도 다른 샤드에 저장되기 때문에 조회가 되지 않습니다.


`member` 인덱스를 생성 후 테스트를 해보면 정상적으로 조회가 됩니다.

```
PUT member/_doc/member_aa?routing=site_a
{
  "site_code": "site_a",
  "member_code": "member_aa",
  "join_fields": "member"
}

PUT member/_doc/member_aa-2024-04-04?routing=site_a
{
  "site_code": "site_a",
  "member_code": "member_aa",
  "connected_at": "2024-04-04",
  "join_fields": {
    "name": "connections",
    "parent": "member_aa"
  }
}



GET member/_search?routing=site_a
{
  "query": {
    "has_child": {
      "type": "connections",
      "query": {
        "match_all": {}
      }
    }
  }
}
```

## 결론
Parent-child 모델링은 Index Rolling 적용이 불가능한 것 같습니다. 부모와 자식이 동일한 샤드에 있어야 한다는 제약으로 `routing_id` 를 지정해도, 다른 인덱스에 있다면 다른 샤드에 저장되기 때문에 조회가 되지 않습니다.


## 그렇다면
샤드가 1개인 경우에는 성공할까 ? 라는 생각이 들었습니다.

```
PUT member-202503
{
  "settings": {
    "refresh_interval": "5s", 
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "site_code": {
        "type": "keyword"
      },
      "member_code": {
        "type": "keyword"
      },
      "connected_at": {
        "type": "date"
      },
      "join_fields": {
        "type": "join",
        "relations": {
          "member": "connections"
        }
      }
    }
  }
}

PUT member-202504
{
  "settings": {
    "refresh_interval": "5s", 
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "site_code": {
        "type": "keyword"
      },
      "member_code": {
        "type": "keyword"
      },
      "connected_at": {
        "type": "date"
      },
      "join_fields": {
        "type": "join",
        "relations": {
          "member": "connections"
        }
      }
    }
  }
}
```

미래의 시점으로 인덱스를 생성하였습니다.
생성 후 보니 `member-202503` 인덱스는 `master2` 노드에, `member-202504` 인덱스는 `master3` 노드에 저장되었습니다.

그렇다면 `member-202505` 인덱스의 Primary Shard 를 `master2` 로 옮긴 다음 다시 테스트를 해보겠습니다. 

Elasticsearch 의 [reroute API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html) 를 이용하여 Shard 를 이동시킬 수 있습니다.

```
POST /_cluster/reroute
{
  "commands": [
    {
      "move": {
        "index": "member-202504", "shard": 0,
        "from_node": "master3", "to_node": "master2"
      }
    }
  ]
}
```

그 후 확인해보면 `master2` 로 이동 된 것을 확인했습니다.

이제 두개의 인덱스 모두 같은 노드의 0번째 Primary 샤드에 저장되었으니 데이터 색인 후 조회가 되는지 확인해보겠습니다.

```
PUT member-202503/_doc/member_aa?routing=site_a
{
  "site_code": "site_a",
  "member_code": "member_aa",
  "join_fields": "member"
}


PUT member-202504/_doc/member_aa-2025-04-04?routing=site_a
{
  "site_code": "site_a",
  "member_code": "member_aa",
  "connected_at": "2025-04-04",
   "join_fields": {
    "name": "connections",
    "parent": "member_aa"
  }
}

GET member-2025*/_search?routing=site_a
{
  "query": {
    "has_child": {
      "type": "connections",
      "query": {
        "match_all": {}
      }
    }
  }
}


{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

```

그래도 조회가 되지 않습니다.


## 마지막 결론
Parent-child 모델링은 같은 인덱스와 같은 샤드에 저장되어야 조회가 가능합니다.


## reference
- https://www.elastic.co/guide/en/elasticsearch/reference/current/parent-join.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html