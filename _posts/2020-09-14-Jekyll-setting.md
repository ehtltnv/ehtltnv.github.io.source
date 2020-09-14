---
layout: post
title:  Jekyll setting
data:   2020-09-14
author: ehtltnv
categories: [ETC]
tags: [Jekyll, git, docker, codeserver]
---

첫 게시물과 Jekyll.

이전부터 `Markdown`에 대해서 조금조금씩 공부했었고, 기술 블로그를 하나 만들어서 공유하면 좋겠다라는 생각에 `Github`에 `Jekyll`를 사용해서 시작하게 됨. 역시나 제대로 하는게 없어서 환경을 구성하는 내용으로 첫 게시물을 작성하게 되었음.

본 게시물에 쓰이는 `Jekyll`와 `codeserver`는 태어나 처음 접하는 것이었고, `Docker`는 그냥 이것저것 해보고 싶어서 혼자 공부해서 진행함. 그렇기 떄문에 자세한 설명은 생략하고 환경 구성을 해서 원하는 기능을 쓸 수 있을 정도를 목표로 글을 작성.

# 1. Jekyll 세팅

## 1.1. Docker for windows

- Docker for windows 설치 
- WSL2 기반 가상화 활성화
- Docker 기반으로 Jekyll를 세팅한 이유
  - Jekyll를 위한 라이브러리들을 설치하기 싫어서
  - 다른 OS에서도 상용할 수 있게 하고 싶어서
  - 로컬에 프로그램 설치를 최소화 하겠다는 다짐이 있어서 

## 1.2. Docker Container 생성

- 여기서는 Docker 사용에 대한 설정은 최소화하고 설치 및 설정에 초점
- ubuntu 최신 버전(20.04) Container 생성

```
<!--
 run : container 생성 및 실행
 -d : 백그라운드 실행
 -it : STDIN 유지, tty 활성화
 --name : container 이름 설정
 -v : 마운트. HOST:CONTAINER
 -p : 포트 바운딩. HOST:CONTAINER
 -->

$ docker run -d -it --name jekyll \
-v c:\blog:/opt/blog \
-p 4000:4000 -p 8000:8000 \
ubuntu /bin/bash
```

- 4000번 포트는 Jekyll 접속용, 8000번 포트는 codeserver 접속용

## 1.3. Jekyll 설치

### 1.3.1. container 접속

```
$ docker exec -it jekyll bash
```

### 1.3.2. Jekyll 설치

```
$ apt update
<!-- ruby 설치 -->
$ apt install -y ruby-full build-essential zlib1g-dev
<!-- jekyll 설치 -->
$ gem install jekyll bundler
$ jekyll -version
4.1.1
```

## 1.4. Jekyll 기본 구성 및 실행

### 1.4.1. 초기화

```
$ cd /opt/blog
<!-- bundler 프로젝트 생성(Gemfile 생성) -->
$ bundle init
<!-- Gemfile에 jekyll 추가 -->
$ bundle add jekyll
<!-- jekyll 초기 세팅 -->
$ bundle exec jekyll new --force --skip-bundle .
<!-- 필요 라이브러리 설치 -->
$ bundle install
```

### 1.4.2. 실행

```
<!-- 
 jekyll serve : 빌드와 함께 서버 실행 
 --livereload : 변경사항 반영
 --host : Listen Address
 --port : 포트. 기본 4000
-->

$ bundle exec jekyll serve --livereload --host 0.0.0.0
```

- http://localhost:4000 접속 확인

---

# 2. codeserver 세팅

## 2.1. 설치

- 위에서 설치한 Jekyll Docker Container에 설치
- 굳이 데이터도 로컬에 공유하면서 서버에 에디터를 설치하는 이유
  - Docker만 있으면 어떤 장비든 상관없이 작업을 할 수 있는 통일된 환경을 세팅하고 싶어서
  - 로컬에 프로그램 설치를 최소화 하겠다는 다짐이 있어서 
  - 신기해서 .. ㅎ
- 로컬에서 수정하겠다면 이 부분은 그냥 넘어가도 괜찮음

```
<!-- docker 접속 -->
$ docker exec -it jekyll bash

$ apt install -y curl vim

$ mkdir /opt/codeserver && cd /opt/codeserver
$ curl -OL https://github.com/cdr/code-server/releases/download/v3.5.0/code-server-3.5.0-linux-x86_64.tar.gz
$ tar zxvf code-server-3.5.0-linux-x86_64.tar.gz
$ ln -s /opt/codeserver/code-server-3.5.0-linux-x86_64/code-server /usr/local/bin/code-server
```

## 2.2. 실행

```
<!-- 
웹에서 codeserver 접속 시 사용하는 비밀번호
나중에 Docker 환경변수로 변경 예정 
-->
$ export PASSWORD=password

<!--
 --host : listen Address
 --port : 포트. 기본 8080
-->
$ code-server --host 0.0.0.0 --port 8000 /opt/blog
```
- http://localhost:8000 접속 확인 

---

# 3. Git 세팅

## 3.1. Git 설치

- 역시 Jekyll Docker Container 안에 설치

```
<!-- Docker 접속 -->
$ docker exec -it jekyll bash

$ apt install -y git
```

## 3.2. Git repository 초기화 및 push

```
$ git config --global user.name [이름]
$ git config --global user.email [이메일 주소]

$ cd /opt/blog
$ git init
$ git add .
$ git commit -m "init"
<!-- 사전에 GitHub에 레파지토리 생성 -->
$ git remote add origin "https://github.com/ehtltnv/ehtltnv.github.io.git"
$ git push -u origin master
```

---

# 4. 실행 스크립트 작성

- systemctl을 사용하려 했으나 docker의 ubuntu에는 기본적으로 systemctl이 없고, 설치해서 사용하려다가 포기 .. 굳이 필요없겠다 싶어서 실행 스크립트를 작성하고 Docker Container 실행 시 호출하는 걸로 정함

## 4.1. 스크립트 생성

```
$ mkdir -p /opt/bin && cd /opt/bin
$ vi startup.sh
#!/bin/bash
code-server --host 0.0.0.0 --port 8000 /opt/blog
```

- 생각보다 Jekyll의 --livereload 옵션이 유용하지 않음. 초기 세팅이라 그런지 구성 변경이 많아서 어차피 재시작 해야해야 해서 스크립트에서 삭제함.
- 스크립트가 별 필요없겠다 싶어짐 .. ㅎ
- 나중에 어느정보 블로그가 안정되면 이 부분은 다시 검토

## 4.2. 실행

```
<!-- 지금까지 구성한 내용을 docker commit -->
$ docker stop jekyll
$ docker commit jekyll ubuntu:jekyll

$ docker rm jekyll

$ docker run -d --name jekyll \
-v c:\blog:/opt/blog \
-p 4000:4000 -p 8000:8000 \
-e PASSWORD=password \
ubuntu:jekyll /opt/bin/startup.sh
```

- Jekyll은 code-server terminal에서 실행

---

# 정리
- 생각보다 어렵다 .. 사실 테마 적용하게 세상 오래 걸림. (아직도 미완)
- Markdown이 생각했던 대로 안 나오는 경우도 많이 있어서 반복적인 수정 작업이 필요.
- codeserver는 보안만 잘 관리하면 좋은 것 같음.
- 어쨌든 시작함. 화이팅.
