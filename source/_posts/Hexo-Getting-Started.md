---
title: Hexo | Getting Started
description: ' '
date: 2024-03-18 22:11:45
categories:
tags:
- Hexo
---

# 建立一個新的 Hexo 網站

## 安裝套件
```shell
$ npm install hexo-cli -g
$ npm install -g npm@10.5.0
```

- 如果要把 Hexo deploy 到 Github 上面，需要先安裝 `hexo-deployer-git`。
  - 相關設定，請參考 [建立自己Blog系列(三) Hexo next theme 介紹](https://ithelp.ithome.com.tw/articles/10257569)。

```shell
$ npm install hexo-deployer-git --save
$ nano _config.yml
```

## 網站初始化
如果沒有設定 folder 的話，Hexo 會在目前的資料夾建立網站。
```shell
$ hexo init myblog
$ cd myblog/
```

## 新增一篇 Post

```shell
$ hexo new "Hexo | Getting Started"
$ vim source/_posts/Hexo-Getting-Started.md
```

## 啟動伺服器
預設是 [http://localhost:4000/](http://localhost:4000/)

```shell
$ hexo server
$ hexo s
```

---
# 部署三部曲

1. 清除快取檔案 (db.json) 和已產生的靜態檔案 (public)
```shell
$ hexo clean
$ hexo cl
```

2. 產生靜態檔案 (部署網站前先產生靜態檔案)
```shell
$ hexo generate
$ hexo g
```

3. 部署網站
```shell
$ hexo deploy
$ hexo d
```

---
# Clone & Run your Repository

## Repository 跑起來
```shell
$ git clone https://github.com/bessyhuang/bessyhuang.github.io.git
$ cd bessyhuang.github.io/
$ git checkout backup
$ npm install
$ hexo server
```

## 備份於 GitHub Repository 的 `backup 分支`
- git checkout
  - 切換分支。Switch branches or restore working tree files

```shell
$ git checkout backup
$ git add --all
$ git commit -m "New function or something modified"
$ git push --set-upstream origin backup
```

---
# References
- [Hexo 指令](https://hexo.io/zh-tw/docs/commands.html)
- [建立自己Blog系列(三) Hexo next theme 介紹](https://ithelp.ithome.com.tw/articles/10257569)
- [CI/CD X Jenkins、CircleCI、Github Action (28)](https://ithelp.ithome.com.tw/articles/10308093)
- [[實作筆記] Hexo CI/CD 設置](https://blog.marsen.me/2022/09/26/2022/Hexo_CICD/)
- [Hexo 備份至 GitHub](https://anemology.cc/post/hexo-backup/)
- [Hexo - 基本設定與 GitHub Page 發佈流程](https://afun.tw/blog/20201114-hexo-init-github-page/)
- [hexo 新增 tag、catagories、about等頁面](https://op30132.github.io/2019/12/24/hexo-tag-page/)
