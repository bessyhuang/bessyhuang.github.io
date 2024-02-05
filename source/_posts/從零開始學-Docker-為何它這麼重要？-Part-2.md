---
title: 從零開始學 Docker - 為何它這麼重要？(Part 2)
description: ' '
date: 2024-02-04 23:59:20
categories:
tags:
---

```
Docker 的三大功用：
1. 簡化部署流程
2. 跨平台部署
3. 建立乾淨測試環境
```

# 功用二：跨平台部署

## 觀念解說

Docker 的強大功能之一是「跨平台部署」！透過把程式及其相依的環境、套件打包成 Docker Image（aka. 程式部署包），你可以輕鬆地在不同作業系統上（Linux、MacOS 或 Windows）運行你的應用程式。

完成 Docker Image 的製作後，成功上傳至 Docker Hub（aka. 存放 Docker Image 的雲端空間），使其他使用者能夠在全球的任何地方、任何作業系統上輕鬆進行下載。隨後，透過在各自作業系統上安裝 Docker Engine，可以協調作業系統與 Docker 容器之間的運行，而無需擔心環境相容性問題。

簡言之，Docker 讓跨平台部署變得快速、便捷，確保了一致的部署體驗。

---
# 實作示範

本章節示範內容：部署一個 Django Website 於不同作業系統上。
藉由撰寫 Dockerfile 來建立 Docker Image，並且將製作好的 Docker Image 上傳至 Docker Hub，以便進行跨平台部署。

## 準備 Django Website

```shell
$ django-admin startproject mysite
$ cd mysite
$ django-admin startapp app1
```

網站的階層架構，如下所示。
```shell
└── mysite/
    ├── app1/
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── apps.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── manage.py
    └── mysite/
        ├── __init__.py
        ├── asgi.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py
```

### mysite/settings.py
```python
#...

INSTALLED_APPS = [
    #...
    'app1',
]
```

### mysite/urls.py
```python
#...
from django.urls import re_path
from app1.views import hello_world


urlpatterns = [
    #...
    re_path(r'^hello/$', hello_world),
]
```

### app1/views.py
```python
#...
from django.http import HttpResponse


def hello_world(request):
    return HttpResponse("Hello, World!")
```

## 撰寫 Dockerfile

Dockerfile 是用於建置 Docker Image 的腳本。

```dockerfile
FROM python:3.10-alpine
ENV myworkdir /home/django_site
WORKDIR ${myworkdir}
COPY ./mysite/ ${myworkdir}

RUN apk update
RUN apk add vim
RUN pip install django

EXPOSE 8000
ENTRYPOINT ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

## 建立 Docker Image

```shell
└── site/
   ├── Dockerfile
   └── mysite/
        ├── app1/
        │    ├── ...
        ├── manage.py
        └── mysite/
             ├── ...
```

cd 到有 `Dockerfile` 的目錄下。接著，建立名為 `django_site` 的 Docker Image。
```shell
$ cd site/ 
$ docker build -t bessyhuang/django_site . --no-cache
```

## 執行

```shell
$ docker run -d -p 8001:8000 django_site 
```
打開瀏覽器 [http://localhost:8001/hello/](http://localhost:8001/hello/)
