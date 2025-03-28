---
layout: post
title: 'Kibana Plugin - Index Last Synced 만들기'
description: 'Kibana Plugin - Index Last Synced 만들기'
date: 2025-03-28
draft: false
author: getsolaris
tags: [kibana, kibana-plugin]
---

## 들어가기 전에

이번에는 Elasticsearch 클러스터에서 지금 어떤 인덱스가 얼마나 오래 됐는지, 데이터 수집이 잘 되고 있는지를 한눈에 확인하고 싶어 만든 Kibana 플러그인 개발기를 소개하려고 합니다.


## 왜 만들게 되었을까?
운영 중인 클러스터에는 수백 개의 인덱스가 존재합니다. 이 중 일부는 데이터 수집이 중단됐거나 지연되고 있었지만, Kibana에서 이를 한 번에 확인할 수 있는 기능은 없었습니다.

기존 방식은 아래와 같았습니다.

- Dev Tools에서 인덱스 하나하나 _search 쿼리로 최근 @timestamp 조회

> "그냥... 한 화면에서 인덱스별로 마지막 수집 시점이 보이면 좋겠다."

이 한마디로 시작된 사이드 프로젝트였습니다.


## Index Last Synced란?
간단히 말해, 이 플러그인은 모든 인덱스의 마지막 동기화 시간(@timestamp)을 한눈에 보여주는 대시보드입니다.


![1](https://raw.githubusercontent.com/getsolaris/kibana-plugin-index-last-synced/refs/heads/main/images/22222.png)

주요 기능
- Elasticsearch 클러스터의 모든 인덱스 리스트 조회
- 각 인덱스에 대해 가장 최근 @timestamp 값을 표시
- YYYY-MM-DD HH:mm:ss 형식 + 상대 시간(예: 5분 전)
- 인덱스 필터링 옵션 제공
- 시스템 인덱스 포함/제외
- 히든 인덱스 포함/제외
- 검색어 필터링
- 인덱스 상태(open/close) 및 문서 수 표시
- 정렬 지원 (이름, 타임스탬프, 문서 수)
- 3초마다 자동 새로고침
- 인덱스 이름 클릭 시 관리 페이지로 이동



## 아쉬운 점 및 개선 방향
1. 현재는 @timestamp 필드가 없거나 다른 필드를 쓰는 경우는 누락됨 
   - 설정 가능한 타임스탬프 필드명을 지원하는 옵션 추가 예정



## 마무리하며
단순하지만 실용적인 도구가 오히려 가장 유용할 때가 많습니다. Index Last Synced는 그런 문제의식에서 출발했습니다.
Elasticsearch와 Kibana를 운영하면서 데이터가 잘 들어오고 있는지 모니터링할 수단이 필요하셨다면, 이 플러그인이 도움이 되길 바랍니다.

GitHub 링크: https://github.com/getsolaris/kibana-plugin-index-last-synced

