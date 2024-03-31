---
layout: post
title: 'Elasticsearch token 과 term 에 대해 정리'
description: 'Elasticsearch token, term, search'
date: 2024-03-31
draft: false
author: getsolaris
tags: [Elasticsearch, token, term]
---


## 들어가기 전에

Elasticsearch 에서 `token` 과 `term` 에 대해 정리합니다.

이 두 용어는 검색 및 인덱싱 시에 중요한 역할을 합니다.

서로 밀접하게 연관되어 있지만, 구체적인 의미와 사용 방법이 다르기 때문에 구분하여 정리하였습니다.

---


## Token

`Token` 은 텍스트 데이터를 인덱싱하기 전에 텍스트를 나누는 단위입니다.

예를 들어 Hello World 라는 문장을 분석기를 통해 처리하면 `Hello` 와 `World` 라는 두 개의 토큰이 생성됩니다.

이 과정을 토큰화라고 하는데, 토큰화 과정은 Analyzer 를 통해 이루어집니다.

> 텍스트를 토큰으로 나누는 것은 Tokenizer 의 역할이지만 이는 Analyzer 의 일부 동작입니다.


## Term

`Term` 은 인덱싱된 문서에서 사용되는 단어를 의미합니다. 인덱스에서 문서를 검색할 때 검색 쿼리는 `Term` 을 기반으로 이루어집니다.


## 인덱싱된 문서를 검색할 때

Elasticsearch 에서는 인덱싱된 문서를 검색할 때 `Term` 을 기반으로 검색을 수행합니다.

사용자가 검색 쿼리를 입력하면, 검색 쿼리는 `Analyzer` 를 통해 토큰화되어 `Term` 으로 변환됩니다.

변횐된 `Term` 을 기반으로 인덱스에서 검색을 수행하게 됩니다.


## reference
- https://www.elastic.co/blog/found-text-analysis-part-1