---
title: 從零開始學 Docker - 為何它這麼重要？(Part 3)
description: ' '
date: 2024-04-02 22:26:07
categories: Docker
tags:
- Docker
- 觀念解說
---

```
Docker 的三大功用：
1. 簡化部署流程
2. 跨平台部署
3. 建立乾淨測試環境
```

# 功用三：建立乾淨測試環境

## 觀念解說



---
# 實作示範

本章節示範內容：建立乾淨的測試環境。
利用 Dockerfile 建立一份客製映像檔（Docker Image），並把測試環境跑起來。

## 準備 MongoDB Container

```shell
$ docker network create test_network
$ docker run -d --name mongodb --network test_network bessyhuang/py3.10_mongo:v1 
$ docker exec -it mongodb bash
```

## 準備 Django Platform Container

```shell
$ docker run -it -p 8001:8000 -v /Users/bessyhuang/Downloads/TEST_mysite/esiot-platform:/home/ --network test_network --name podman_platform python:3.11.8-slim bash
# apt update
# apt install gcc python3-dev -y
# pip install -r py3.11.8_requirements.txt
# python manage.py runserver 0.0.0.0:8000
```



```dockerfile
FROM python:3.11.8-slim
MAINTAINER Bessy

COPY ./mysite ./home/
WORKDIR /home/mysite
RUN apt-get update && apt-get install iproute2 -y && apt-get install vim -y

EXPOSE 8000
ENTRYPOINT ["python", "manage.py", "runserver"]
```


Example
```
FROM alpine:latest
ENV myworkdir /var/www/localhost/htdocs/
ARG whoami=Bessy
WORKDIR ${myworkdir}
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
RUN echo "<h3>I am ${whoami}. I am taking this great Docker Course. Round 01</h3>" >> index.html
RUN echo "<h3>I am ${whoami}. I am taking this great Docker Course. Round 02</h3>" >> index.html
RUN echo "<h3>I am ${whoami}. I am taking this great Docker Course. Round 03</h3>" >> index.html
COPY ./content.txt ./
RUN ls -l ./
RUN cat ./content.txt >> index.html
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```
