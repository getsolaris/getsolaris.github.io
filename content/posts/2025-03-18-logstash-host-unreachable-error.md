---
layout: post
title: 'Logstash HostUnreachableError 이란 무엇인가 ?'
description: 'Logstash HostUnreachableError 에러에 대해 알아보고 problem solving'
date: 2025-03-18
draft: false
author: getsolaris
tags: [Elasticsearch, Logstash]
---

현 직장에 다니면서 ELK 스택을 운영하고 있다.

어느날 Logstash 에서 `HostUnreachableError` 가 발생했다.

```java
[WARN ] 2024-12-11 01:40:35.567 [[logstash]>worker0] elasticsearch - Marking url as dead. Last error: [LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError] Elasticsearch Unreachable: [http://node15:9200/_bulk][Manticore::SocketTimeout] Read timed out {:url=>http://node15:9200/, :error_message=>"Elasticsearch Unreachable: [http://node15:9200/_bulk][Manticore::SocketTimeout] Read timed out", :error_class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError"}
[ERROR] 2024-12-11 01:40:35.568 [[logstash]>worker0] elasticsearch - Attempted to send a bulk request but Elasticsearch appears to be unreachable or down {:message=>"Elasticsearch Unreachable: [http://node15:9200/_bulk][Manticore::SocketTimeout] Read timed out", :exception=>LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError, :will_retry_in_seconds=>32}
[WARN ] 2024-12-11 01:40:40.102 [Ruby-0-Thread-12: /usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-output-elasticsearch-11.4.1-java/lib/logstash/outputs/elasticsearch/http_client/pool.rb:213] elasticsearch - Restored connection to ES instance {:url=>"http://node15:9200/"}
.
.
.
```

## 오류 분석

```java
[ERROR] 2024-12-11 01:40:35.568 [[logstash]>worker0] elasticsearch - Attempted to send a bulk request but Elasticsearch appears to be unreachable or down {:message=>"Elasticsearch Unreachable: [http://node15:9200/_bulk][Manticore::SocketTimeout] Read timed out", :exception=>LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError, :will_retry_in_seconds=>32}
```

우선 Logstash 에서 ES 으로 Bulk 요청을 보내는 과정에서 ES 에서 처리를 못한 것으로 보인다.

오류 메시지를 보면 Manticore::SocketTimeout이 발생했으며, 이는 Logstash가 일정 시간 동안 Elasticsearch의 응답을 기다렸지만, 결국 응답을 받지 못하고 타임아웃이 발생한 상황이다.


## 발생 원인 분석
1. **Elasticsearch의 과부하**
2. 네트워크 문제 (가능성 낮음, 확인 필요)
3. Elasticsearch의 클러스터/노드 장애 (가능성 낮음, 확인 필요)


## 클러스터/노드 상태 확인
해당 오류가 발생한 뒤 클러스터의 상태를 확인했다.
클러스터의 상태는 Green, 모든 노드들의 상태도 Green 으로 양호했다.

그렇다면 노드의 thraed pool 을 확인 해볼 차례이다.
```
curl -XGET "http://node15:9200/_cat/thread_pool?v=true"
```

예를 들어
- queue 값이 100~200 이상이면 대기 중인 요청이 많다는 의미, 즉 Elasticsearch가 현재 들어오는 Bulk 요청을 즉시 처리하지 못하고 대기열에 쌓고 있음.
- rejected 값이 높으면 Elasticsearch가 새로운 요청을 거부하는 상태임.
→ 이는 Elasticsearch가 처리 가능한 범위를 초과하여 더 이상 Bulk 요청을 받을 수 없는 상황임.


## 해결 방법
위 분석 결과를 바탕으로, Elasticsearch가 과부하 상태였음을 확인했다.
이를 해결하기 위해, Bulk 요청 크기를 조정하는 방식(1번 방법)을 적용했다.

Logstash의 pipeline.batch.size 값을 줄여서 한 번에 처리하는 요청 크기를 낮춤.

```
# logstash.yml

# 기존
pipeline.batch.size: 15000

# 변경
pipeline.batch.size: 5000
```

이 조정을 통해 Elasticsearch가 Bulk 요청을 보다 원활하게 처리할 수 있었고, 이후 오류가 발생하지 않음을 확인하였다


## 추가 공부
Elasticsearch 의 7버전 이상부터 queue_size 의 기본 값은 10000 이다.
```
For single-document index/delete/update and bulk requests. Thread pool type is fixed with a size of # of allocated processors, queue_size of 10000. The maximum size for this pool is 1 + # of allocated processors.
```

6.x 버전에서는 queue 크기가 200이었음을 확인
7.x 버전에서는 Bulk Thread Pool의 queue 크기가 10000이므로, 대기열에 10000개의 요청이 차면 그 이후 요청부터 rejected가 증가한다.

예시: queue_size(10000)를 초과한 100001번째 요청부터 rejected가 증가함

## Reference
- [Elasticsearch Thraed pools] https://www.elastic.co/guide/en/elasticsearch/reference/7.17/modules-threadpool.html
- [Elasticsearch CAT API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-thread-pool.html)
- https://discuss.elastic.co/t/write-thread-pool-queue-size-defaults/277449