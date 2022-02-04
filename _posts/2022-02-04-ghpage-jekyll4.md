---
layout: post
title: 在 Github Pages 上使用 Jekyll 4
date: 2022-02-04 19:46 +0800
category: update
tag: jekyll deploy
---

真的是要累鼠...，才剛寫完一篇 Vim 想說加點酷酷ㄉ功能，結果馬上給我跑一堆問題出來：））

### 奇怪的小問題：CJK word count
簡單來說就是想在這個部落格加上跟 Medium 一樣的 Reading time helper（上面 👀 n mins 的部分），雖然有查到最簡單的算法就是總字數除以每分鐘的閱讀字數，但就遇到英文跟中文的字數算法不一樣的問題，因為英文的字數算法是空格隔開，但中文的算法是以一個 character 為一個字。不過在本地端開發的時候有查到 Jekyll 4.0 之後的 `number_of_words` 有支援 `cjk`（中文、日文、韓文）的字數計算，於是快樂的使用 `{{ "你好hello世界world" | number_of_words: "auto" }}` 的語法，誰知道推上 Github 之後發現哇，Github Pages 只支援到 Jekyll 3.9 😀🔪，於是這篇文就誕生了 😀🔪🔪🔪🔪

### 使用 Github Action 來部署 Jekyll site
基本上我是照著[官方的文件](https://jekyllrb.com/docs/continuous-integration/github-actions/)做的，其實文件上說的都很清楚詳細。簡單來說就是在現有的 Jekyll 專案裡新增一個 `.github/workflow/<your-action>.yml` 的檔案，目的是要在 Github 的 repo 上新增 Action，最基本的設定如下：

```yml
name: Build and deploy Jekyll site to GitHub Pages

on:
  push:
    branches:
      - main # or master before October 2020

jobs:
  github-pages:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/cache@v2
      with:
        path: vendor/bundle
        key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile') }}
        restore-keys: |
          ${{ runner.os }}-gems-
      - uses: helaili/jekyll-action@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

這些動作的設定基本上跟在寫 docker-compose 還有 gitlab-ci 是一樣的（畢竟是一樣的東西），官網上也都有說明每一行在幹嘛，這邊就不贅述了，比較重要的地方是最後一個 step 使用了 Jekyll 專用的 action `helaili/jekyll-action@v2` 。

在新增完這個檔案後，只要推上 Github 之後基本上就沒問題了，推完以後可以去 repo 的 Action tab 檢查這個剛設定的 action 有沒有確實跑起來並且成功跑完。基本上跑完就不會有錯了，不過我也是在這裡遇到了第三個大坑（這個坑最白癡）。

#### 修改 Source Branch
官方文件上說，這個 action 執行完之後會把所有 build 出來的內容放到 `gh-pages` 的分支裡面，也不需要自己新增 `gh-pages` 的分支，並且在 Action tab 裡我自己新增的那個 action 是成功執行完畢的，不過我怎麼看 github 的介面都顯示 your site is ready to be published(aka 你的 action 掛了)、還有這個網站都沒有更新過後該要有的樣子。

最後我仔細去檢查 jekyll action 執行時的訊息，才發現原來這個 action 是把 build 好的東西丟到 master branch 上，而不是 gh-pages branch 😀🔪🔪🔪

![](/assets/img/action-log.png)

最後的最後，去到 Github repo > Settings > Pages > Source Branch，改成剛剛在 jekyll action log 上看到的分支，就能看到 build 好ㄉ、4.2.0 版的 Jekyll site 了，是不是其實真的很簡單呢😀🔪🔪🔪

<font color="grey" style="font-size: 24px">Reference</font>
* [Jekyll Read Time without Plugins](https://int3ractive.com/blog/2018/jekyll-read-time-without-plugins/)
* [Github Action(Jekyll Docs)](https://jekyllrb.com/docs/continuous-integration/github-actions/)
