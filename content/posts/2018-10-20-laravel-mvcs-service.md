---
layout: post
title: 'Laravel Service Layer 생성 라이브러리 개발'
description: '첫 회사에서 경험하고 라이브러리로 만들기'
date: 2018-10-20
author: getsolaris
tags: [Laravel]
---


Laravel은 기본적으로 MVC (Model-View-Controller) 패턴을 사용하여 구조화되어 있습니다.

하지만, 회사에 새로 합류하여 프로젝트를 살펴보니 MVCS (Model-View-Controller-Service) 패턴을 이용하고 있었습니다.

![MVC 패턴](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/1da80aee3f4055d3dda05ef16068b0dbf6bb75a5/assets/images/post/laravel-mvcs/mvc.jpg)
> MVC 패턴의 일반적인 흐름입니다.

MVC 패턴은 클라이언트로부터 요청이 오면 Controller가 그 요청을 받아 Model을 통해 데이터를 처리하고, 그 결과를 View를 통해 사용자에게 보여주는 방식으로 동작합니다.

![MVCS 패턴](https://raw.githubusercontent.com/getsolaris/getsolaris.github.io/1da80aee3f4055d3dda05ef16068b0dbf6bb75a5/assets/images/post/laravel-mvcs/mvcs.png)
> MVCS 패턴의 작동 방식입니다.

MVCS 패턴은 이와 다르게, Controller는 Service에 의존성을 주입 (Dependency Injection) 받고, Service에서 비즈니스 로직을 처리하도록 하여, Controller와 Model 사이의 결합도를 낮춥니다.

이 MVCS 패턴을 사용하니, 개인 프로젝트에서도 구조가 더 깔끔하게 정리되어 유지보수가 쉬워진 경험이 있었습니다.

Laravel의 Artisan 명령어 중에서 Controller를 생성하는 명령는 있지만,
```shell
php artisan make:controller PostController
```

Service Layer를 생성하는 명령어는 별도로 제공되지 않아, 서비스 클래스를 수동으로 생성해야 했습니다.

이런 수고를 덜기 위해, 사용자 정의 Artisan 명령어를 만들어 서비스 클래스를 쉽게 생성할 수 있도록 했습니다.

```shell
php artisan make:service PostService
```



## download
- [https://github.com/getsolaris/laravel-make-service-command](https://github.com/getsolaris/laravel-make-service-command)

## reference
- MVS 이미지 : https://blog.cloudboost.io/what-is-model-view-controller-124a9942246
- MVCS 이미지 : https://glossar.hs-augsburg.de/Model-View-Controller-Service-Paradigma
