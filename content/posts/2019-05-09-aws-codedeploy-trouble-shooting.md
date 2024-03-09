---
layout: post
title: 'AWS CodeDeploy 파일 존재 배포 실패 에러 발생시'
description: 'The deployment failed because a specified file already exists at this location'
date: 2019-05-09
author: getsolaris
tags: [AWS]
---

## Introduction

> The deployment failed because a specified file already exists at this location

AWS CodeDeploy 로 배포 시 `codedeploy-agent` 서버에 동일한 파일이 존재할 경우 위와 같은 에러가 발생합니다.
<br>
라라벨 프로젝트를 자동 배포를 하기위해 TravisCI 를 이용해 CodeDeploy 로 배포를 진행 하려고 했는데 위와 같은 충돌에 해결법을 찾았습니다.
<br>
참고로 CodeDeploy는 파일 덮어쓰기(`overwrite`)가 불가능 하다는것을 아래 래퍼런스 링크에서 확인 가능합니다. (AWS CodeDeploy GUI 수동 배포시 덮어쓰기 옵션 선택이 가능합니다.)


## Solution

배포 시 release 디렉터리에 배포 후 appspec hook 을 통해 스크립트를 실행 시킵니다.
<img src="https://docs.aws.amazon.com/codedeploy/latest/userguide/images/lifecycle-event-order-ecs.png">
> 출처: AWS CodeDeploy appspec lifecycle

<br>
빠른 이해를 위해 실제로 해결 했던 스크립트를 아래에 첨부 합니다.
<br>

`afterInstall.bash` 를 실행 하게 되면 모든 파일을 제거하므로 반영에서 제외할 파일들은 따로 `release-files` 라는 디렉터리를 생성하여 넣어두고 `copy` 하는 형식을 추가했습니다.

```yml
# appspec.yml

version: 0.0
os: linux
files:
  - source: ./
    destination: /var/www/release/
permissions:
  - object: /var/www/laravel-central-api/
    owner: www-data
    group: www-data
hooks:
  BeforeInstall:
    - location: scripts/beforeInstall.bash
  AfterInstall:
    - location: scripts/afterInstall.bash
```

```bash
# beforeInstall.bash

#!/usr/bin/env bash
if [ -d /var/www/release ]; then
    sudo rm -rf /var/www/release
fi
sudo mkdir -vp /var/www/release
```

```bash
# afterInstall.bash

#!/usr/bin/env bash

sudo rsync --delete-before --verbose --archive /var/www/release/ /var/www/laravel-central-api/ > /var/log/deploy.log

cd /var/www/laravel-central-api
sudo composer install
sudo cp /var/www/release-files/.env /var/www/laravel-central-api
sudo cp /var/www/release-files/Controller.php /var/www/laravel-central-api/app/Http/Controllers
sudo php artisan key:generate
sudo php artisan config:cache
sudo chmod -R 775 storage bootstrap/cache storage/framework storage/logs
sudo chown -R $USER:www-data storage
sudo chown -R $USER:www-data bootstrap/cache
sudo pkill -9 -ef bot.php
nohup php bot.php > /dev/null 2> /dev/null < /dev/null &

if [ -d /var/www/release ]; then
    sudo rm -rf /var/www/release
fi
```

## Reference
[https://github.com/aws/aws-codedeploy-agent/issues/14](https://github.com/aws/aws-codedeploy-agent/issues/14)