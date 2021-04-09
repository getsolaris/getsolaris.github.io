---
layout: post
title: '챗봇으로 일상을 바꾸는 팀에서 백엔드 개발자로 활동하며 개발은 어떻게 진행하는가'
description: '500,000명 이상의 사용자를 보유하고 있는 디스코드 챗봇 에서 1년넘게 백엔드 개발자로 활동하며..'
comments: true
date: 2020-06-24
author: 김민근
tags: [Laravel, php, aws, architecture]
---


## 소개

저는 [팀 크레센도](https://team-crescendo.me)에서 백엔드 개발자로 활동하고 있는 주니어 개발자입니다.

팀 크레센도는 500,000명 이상의 사용자를 보유하고 있는 디스코드 챗봇 운영팀입니다. 지금도 80,000개의 디스코드 서버 문화를 팀 크레센도가 만들어가고 있습니다.

저를 포함하여 팀원 모두가 10대와 20대 초반으로 구성되어 있습니다.

회사에 재직하며 여러 사람들과 협업을 하고자 팀에 가입을 하게되었고, 현재는 어느정도 규모가 있는 서비스를 총 관리까지 하게 되었습니다.

저는 해당 팀에 면접을 보고 2018년 12월에 입단하게 되었습니다. _(면접이 되게 험난했던 기억이..)_

![1. 웃픈 이미지 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/1.png)

이러한 회사도 아닌 작은 팀에서 1년 반 넘게 활동하며, 작은 팀에서는 어떤식으로 협업, 인프라 구성, 개발, 배포와 에러 관리와 마지막으로 서버 모니터링 도구에 대해 작성 해봤습니다.


## 의문점

이러한 챗봇 개발팀에서 어떠한 백엔드 서버를 개발 하나? 라는 의문점을 가질 수 있습니다.

저는 이곳에서 `포르테` API를 개발 하고 있습니다.

__포르테란?__ 결제 대행사 Xsolla(이하 "엑솔라") 그리고, 팀 크레센도 봇들 사이에서 통용되는 데이터를 통합해서 관리하는 API 서버를 의미합니다.

`포르테`는 팀 이름(크레센도)에서 볼 수 있다시피 음악 강약기호에서 파생되었습니다.
![2. 음악 강약기호 이미지 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/2.gif)

> 출처: 신나는 오케스트라

----

# 목차
- [협업을 시작해보자](#협업을-시작해보자-)
  - [협업을 하기 위한 과정](#협업을-하기-위한-과정)
  - [포르테팀에서는 할 일 관리를 어떤식으로 하는가 ?](#포르테팀에서는-할-일-관리를-어떤식으로-하는가-)
    - [우리는 이슈를 이렇게 사용하고 있어요.](#우리는-이슈를-이렇게-사용하고-있어요)
      - [깃허브 라벨링](#알록달록-라벨-무엇일까-)
      - [깃허브 프로젝트](#미니-kanban-프로젝트)
  - [협업을 위한 몇가지의 규칙](#협업을-위한-몇가지의-규칙)
- [개발](#개발)
  - [개발 구성은 어떻게 ?](#개발-구성은-어떻게-)
  - [아키텍처(서버) 구성을 살펴보자](#아키텍처서버-구성을-살펴보자)
  - [코드 테스트 그리고 자동 배포까지 (CI/CD)](#코드-테스트-그리고-자동-배포까지-cicd)
  - [에러를 효과적으로 추적하기](#에러를-효과적으로-추적하기)
- [서버 모니터링 도구](#서버-모니터링-도구)
  - [go access](#go-access)
  - [Grafana](#grafana)


# 협업을 시작해보자 !

## 팀에서 협업이란 ?

저희 팀은 온라인팀으로써 디스코드 플랫폼을 통해 소통, 그리고 매주 일요일 7시 30분마다 회의를 진행하며 각 부서별로 이슈/논의를 진행 해요

__온라인__ 팀으로 각자 개인의 시간에 따라 참여하다 보니 협업이 그렇게 쉽지많은 않았어요.


## 협업을 하기 위한 과정

협업을 손 쉽게 하기 위해 여러가지 도구를 도입을 했어요.

각 부서마다 사용하는 협업 도구는 다르지만, 모든 팀원은 전체적으로 문서 공유를 위해 노션(Notion)을 사용 해요 !


2020년 06월 17일 기준 24명의 팀원과 함께 하고 있습니다. 

서로의 식별을 위해 우리는 뱃지 개념의 '역할'을 사용해요.

![4. 역할 이미지 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/4.png)

저의 담당 직무는 `개발`이고, 프로젝트(부서)는 `포르테(FORTE STAFF)`에 속해 있어요.


## 포르테팀에서는 할 일 관리를 어떤식으로 하는가 ?

`포르테`팀은 전원 모두가 개발자이기 때문에, 할 일 관리는 깃허브의 이슈를 사용하게 되었어요.


### 우리는 이슈를 이렇게 사용하고 있어요.

![5. 이슈 이미지 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/5.png)

우리 `포르테 개발팀`은 할 일(Task)을 모두 깃허브 이슈에 등록합니다.

![6. 이슈 등록 이미지 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/6.png)

기본 양식으로 손 쉬운 할 일을 등록하기 위해 이슈 템플릿을 사용하고 있어요.

우리는 서로가 바쁜걸 알고 있고, 시간을 존중하기 때문에 압박으로부터 벗어나고자 마일스톤(데드라인)을 등록하지 않아요 !

서로를 믿고 개발하기에 데드라인이 없어도, 일정 관리가 잘 되고 있답니다 :)

### 알록달록 라벨, 무엇일까 ?

깃허브에서 기본적으로 제공하는 라벨을 사용하고 있어요

라벨 양식은 오픈소스 프로젝트인 Elastic을 참고했어요.

![7. 라벨 이미지 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/7.png)

라벨을 사용하면 신규 개발인지, 핫픽스 수정, 또는 버전 관리를 할 때 `v1`, `v2` 라벨과 같이 어떤 버전에 속한 이슈인지 한 눈에 알 수 있어요.

### 미니 Kanban, 프로젝트

우리는 유명한 칸반보드 서비스 대신 깃허브의 프로젝트를 사용해요.

깃허브 프로젝트를 도입한지는 별로 안되었어요.. (더 빨리 도입할껄 ㅠㅠ)

![8. 프로젝트 사진 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/8.png)

위 사진처럼 이슈를 바로 확인 가능하고, 어느 프로세스에 머물러있는지 확인이 가능해요.

또한 프로젝트에서 바로 Issue 를 불러올 수 있어요 !

Issue 에 링킹된 Pull Reqeust 도 볼 수 있답니다 !

## 협업을 위한 몇가지의 규칙

깃허브로 모든걸 해결하다 보니 몇가지 규칙이 있어요.

가장 기본적인걸 소개시켜드리자면 `커밋 규칙(Rule)`을 아래와 같이 정해뒀어요.

#### Commit Message Type
- 기능: 기능 추가, 삭제, 변경
- 버그: 버그 수정
- 리팩토링: 코드 리팩토링
- 형식: 코드 형식, 정렬, 주석 등의 변경 (코드 수정, 하지만 동작에 영향은 없는 코드)
- 테스트: 테스트 코드 추가, 삭제, 변경 등
- 문서: 문서 추가, 삭제, 변경
- 기타: 위 여섯가지에 해당되지 않는 모든 변경 (ex: 배포 스크립트 변경)을 포함

#### Commit Message Configuration

```
Type: Title

ex) 기능: 포르테 스토어 서비스단 구현
ex) 기능: 포르테 API 요청에 관련된 로그 출력 기능 구현
ex) 형식: UserController store 메서드 주석 수정
ex) 문서: 리드미 인프라 부분 수정 
```

#### Issue Tracking Commit Type
- 해결: 이슈 해결 시 사용
- 관련: 이슈 해결되지 않은 경우
- 참고: 참고할 이슈가 있을 때 사용

#### Issue Tracking Commit Configuration

```
Type: #n Title
ex) 해결: #54 포르테 스토어 청약철회 에러 수정
```

## 개발

![9. 아키텍처 이미지 첨부](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/9.png)

### 개발 구성은 어떻게 ?

PHP 프레임워크인 Laravel을 도입 하게되었어요. 

처음 개발할 때 5.6 버전이였는데 글 작성일 기준으로 라라벨의 최신 버전은 7.16 ..

![10. 모던퍼그 라라벨 릴리즈 이미지 첨부 후 이것 또한 내가 만든거다 라고 소개](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/10.png)

누구나 알법한 APM (Apache2 + PHP + MySQL) 로 구성했어요 !

### 아키텍처(서버) 구성을 살펴보자

`포르테`의 인프라는 모두 AWS 제품으로 구성되어 있어요 !

- AWS EC2 (Cloud Computer, 포르테의 서버를 맡고 있어요)
- AWS S3 (Forte와 Xsolla 이미지 처리 및 CodeDeploy Build 파일들)
- AWS CodeDeploy (자동배포 서비스)
- ~~AWS RDS~~
- AWS CloudWatch (AWS 모니터링 서비스)

### 코드 테스트 그리고 자동 배포까지 (CI/CD)

`포르테팀`에서는 테스트 코드를 직접 짜고 있지는 않아요. 하지만 이번에 팀 회의를 거쳐 `테스트 주도 개발(TDD)` 를 도입 하기로 했어요 !

코드 테스트는 도입한지 별로 안되어서 이야기 드릴게 없어요!
 
지금은 자동 배포에 대해 이야기를 드릴게요 !

![14. code deploy](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/14.png)

코드 자동 배포 프로세스는 위 사진과 같이 인프라를 구성했어요 !

정리하면, `포르테 개발팀`에서 master 브랜치에 Commit & Push 를 하게 되면 TravisCI 에서 Build 를 진행 하고 만약, Build 가 성공적으로 된다면 AWS S3 버킷에 소스코드가 업로드가 되어요.

### 에러를 효과적으로 추적하기

![12. sentry tracking](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/12.png)

에러를 효과적으로 추적하기 위해 [sentry](https://sentry.io) 서비스를 이용해요. 위 사진처럼 어느 부분에서 해당 이슈가 발생했는지 추적이 쉽게 가능해요.

![13. sentry language](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/13.png)

또한, 많은 플랫폼을 지원하고 있어요 !

에러 트래킹이 필요하다면 이 서비스를 직접 사용해보는 것도 나쁘지 않은 것 같아요, 설치 마저 쉽답니다 :)

## 서버 모니터링 도구

팀 크레센도 `포르테 개발팀`에서 서버 모니터링을 위해 사용하는 두가지의 도구가 있는데, 그 두가지를 소개시켜드리고 해당 포스트를 마치고자 해요.

### Go Access

![15. goaccess image](https://camo.githubusercontent.com/d8bb8f24fd472b605c37509b4520b0eacb96efb5/68747470733a2f2f676f6163636573732e696f2f696d616765732f676f6163636573732d7265616c2d74696d652d68746d6c2d67682e706e673f32303230303531373031)

`GoAccess` 는 오픈소스 웹 로그 분석툴로 쉽게 웹서버의 로그를 분석할 수 있어요.

무엇보다 설치가 되게 쉽기도 해요.

### Grafana

![16. grafana](https://grafana.com/static/img/grafana/showcase_visualize.jpg)

Grafana는 시계열 매트릭 데이터를 시각화 하는데 가장 최적화된 대시보드를 제공해주는 오픈소스에요 !

`포르테` 팀원 중 한분께서 회사에서 그라파나와 프로메테우스(Prometheus) 조합을 사용중인데, 좋다고 말씀 주셔서 포르테에도 도입을 하게 되었어요.

포르테팀도 똑같이 그라파나와 프로메테우스 조합으로 대시보드를 구성 했어요.

![16. prometheus](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/master/assets/images/post/team-crescendo-forte-memoirs/16.png)

프로메테우스의 모듈중 하나인 node_exporter가 정보를 수집, 위 사진과 같이 Endpoint를 열어서 프로메테우스가 데이터를 수집해갈 수 있도록 해줘요.

프로메테우스의 Metric을 이용하여 정보를 시각화할 수 있어요.


## 마무리

팀 크레센도에 대해 궁금하신 점이 있다면 댓글 남겨주세요 ! 

- [팀 크레센도 홈페이지](https://team-crescendo.me)
- [포르테 깃허브](https://github.com/team-crescendo/laravel-forte-api)

감사합니다.
