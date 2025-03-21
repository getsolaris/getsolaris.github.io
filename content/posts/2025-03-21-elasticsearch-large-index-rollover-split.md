---
layout: post
title: 'Elasticsearch 대용량 인덱스 롤오버 테스트 및 인덱스 분할'
description: 'Elasticsearch 대용량 인덱스 롤오버 테스트 및 인덱스 분할'
date: 2025-03-21
draft: false
author: getsolaris
tags: [Elasticsearch, rollover, ilm]
---

## 들어가기 전에

1억 건이 넘는 문서를 포함하고 있으며, 데이터 크기가 6TB에 달하는 인덱스가 있었다.
이로 인해 검색 및 색인 시 Elasticsearch 클러스터에 과부하가 발생하고 있었으며,
특히 해당 인덱스의 부하가 다른 인덱스들의 성능에도 악영향을 미치고 있었다.

색인 시 부하를 줄이기 위해 다음과 같은 조치를 시도했으나, 기존 인덱스는 더 이상 유지하기 어렵다고 판단되었다.

1. Logstash 파이프라인 스케줄을 10초 → 60초로 변경하여 색인 주기 조정
2. 인덱스의 refresh_interval 값을 5초 → 60초 → 180초 → 3600초로 순차적으로 증가
3. Elasticsearch 일부 노드를 스케일업하여 처리 성능 향상 
   - 일부 노드를 스케일업 했지만 부하는 그대로 -> 근본 원인인 게시글 인덱스를 개선


그러나 이러한 조치만으로는 근본적인 문제 해결이 어렵다고 판단되어,
인덱스 롤오버 및 인덱스 분할 전략을 테스트해보기로 했다.

## ILM Rollover 정책 테스트
색인 크기에 따른 부하를 줄이기 위해 용량 기반 Rollover(30GB 단위)를 적용하는 방안을 테스트했다.

### 클러스터의 Life Cycle 수명 주기 변경

기본적으로 ILM 정책은 10분(10m)마다 실행되지만, 더 빠르게 적용되도록 10초(10s)로 변경했다.

```
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "10s"
  }
}
```

### ILM 정책 작성 (30GB 기준 Rollover 적용)
```
PUT _ilm/policy/test-rollover-ilm-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "30gb"
          }
        }
      }
    }
  }
}
```

- hot phase: 인덱스가 활성 상태에서 색인/검색이 이루어지는 단계
- max_size: 인덱스 크기가 30GB를 초과할 때 새로운 인덱스로 자동 롤오버

### Index Template 생성

```
PUT _template/test-rollover
{
  "index_patterns": ["test-rollover-*"],
  "settings": {
    "index.lifecycle.name": "test-rollover-ilm-policy",
    "index.lifecycle.rollover_alias": "test-rollover",
    "number_of_shards": 18,
    "number_of_replicas": 1
  }
}
```

- index_patterns: test-rollover-* 패턴을 따르는 인덱스에 정책 적용 
- index.lifecycle.name: 위에서 생성한 ILM 정책 사용 
- index.lifecycle.rollover_alias: Rollover 발생 시 사용할 인덱스 별칭 지정

### 초기 인덱스 및 alias 설정

```
PUT test-rollover-000001
{
  "aliases": {
    "test-rollover": {
      "is_write_index": true
    }
  }
}
```

## Rollover 테스트 중 발생한 문제
Elasticsearch의 Rollover 기능을 적용했을 때, 롤오버 후 기존 문서 업데이트가 정상적으로 동작하지 않는 문제가 발생했다.


![1](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/main/images/2025-03-21-elasticsearch-large-index-rollover-split/1.png)

test-000001 인덱스에 document_id: p123123 문서가 삽입됨.

![1](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/main/images/2025-03-21-elasticsearch-large-index-rollover-split/2.png)

데이터가 증가하면서 Rollover 발생, 새로운 인덱스 test-000002가 생성됨.

이때 alias 는 동일하지만 is_write_index 가 false로 변경되어 새로운 인덱스로 색인이 이루어짐.

![1](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/main/images/2025-03-21-elasticsearch-large-index-rollover-split/3.png)
이후 test-000003 인덱스가 추가로 생성됨. 

document_id: p123123 문서를 업데이트하려고 하면, 기존 test-000001 인덱스에서 업데이트되지 않고 새로운 test-000003 인덱스에 새로운 문서로 삽입됨.

결과적으로 중복 문서가 생성되면서 데이터 정합성이 깨지고, 검색 성능 저하로 이어졌다.

## ILM의 Rollover 를 하면서 배운 점
ILM의 Rollover 기능은 로그성 데이터처럼 업데이트가 발생하지 않는 인덱스에만 적합하다.하지만 게시글 데이터는 기존 문서의 업데이트가 빈번하게 발생하기 때문에, Rollover 적용 시 정상적인 업데이트가 불가능했다.


## Index Template + MOD 기반 인덱싱 전략
ILM Rollover를 포기하고, Index Template을 활용하여 특정 네이밍 패턴을 가진 인덱스를 자동 생성하도록 변경했다.

- 새로운 인덱스는 test-2025-N 형식으로 생성됨. 
- 색인 시 인덱스를 명시적으로 test-2025-N으로 지정하여 문서 업데이트가 항상 동일한 인덱스에서 이루어지도록 강제.

### MOD 연산을 이용한 인덱스 분할
인덱스 관리를 효율적으로 하기 위해 MOD 연산을 사용하여 특정 인덱스에 색인되도록 설정했다.

```javascript
// 이해를 돕기 위해 간단한 예시로 설명
const indexNumber = document_id % 5; // 5개의 인덱스로 분할
const indexName = `test-2025-${indexNumber}`;
```

- 문서의 ID 값을 특정 숫자로 나눈 나머지를 기준으로 인덱스를 지정
- 예를 들어, MOD(document_id, 70) 값을 사용하면 70개의 인덱스로 분산할 수 있음 
- 같은 document_id는 항상 동일한 인덱스로 매핑되므로 업데이트 시에도 일관성 유지


## 최종
ILM의 Rollover 기능은 업데이트가 발생하는 데이터에는 적합하지 않다.로그성 데이터가 아니라면, Index Template과 MOD 연산을 이용한 인덱싱 전략이 더 효과적이다.


이를 통해 Elasticsearch의 성능을 최적화하면서 데이터 정합성을 유지할 수 있었다.

## Reference
- https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-rollover.html
- https://lasel.kr/archives/604
- https://discuss.elastic.co/t/ilm-rollover-not-working-on-max-size/287999