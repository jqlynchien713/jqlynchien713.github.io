---
layout: post
title: Jekyll + Utterances 實作 gh-pages 留言功能
date: 2022-02-23 14:48 +0800
tags: update jekyll
---

寫部落格沒有留言的功能好像怪怪ㄉ，上課的時候跟朋友討論研究了一下，發現了 Utterances 這個酷工具可以跟 Jekyll & gh-pages 整合，不用用到資料庫就可以實作出留言ㄉ功能了！

## 運作原理
基本上這個外掛是將 github repo 的 issues 整合到靜態網頁本身上，所以當其他使用者在留言時，需要登入 github 的帳戶，並且在其他使用者留言後，這些留言就會更新到 github repo 上的 issues 分頁

這是部落格留言後的樣子
![](/assets/img/comment-demo.png)

留言會同步更新到 github issue
![](/assets/img/comment-gh.png)

## Configuration

### Github 端設定
* 需要先在 Github 上 install utterances app，並且在個人設定頁面設定選擇要套用 utterance app 的 repo
  ![](/assets/img/comment-setup.png)

* 接下來可以複製 Utterances 給的 script，在 `[ENTER REPO HERE]` 填入想要採用 Utterances 的 `UserName/RepoName`。另外 theme 的部分可以到 Utterances 的官網上看，總共有九個主題可以選，我最後選的是跟現在網站比較搭的 github-light

  ```html
  <script src="https://utteranc.es/client.js"
          repo="[ENTER REPO HERE]"
          issue-term="pathname"
          theme="github-light"
          crossorigin="anonymous"
          async>
  </script>
  ```

### Jekyll 端設定
* 先設定好 `config.yml`，指定留言使用的外掛為 Utterances

  ```yml
  # Set which comment system to use
  comments:
    # 'disqus' or 'utterances' are available
    provider:            utterances

  # You must install utterances github app before use.(https://github.com/apps/utterances)
  # Make sure all variables are set properly. Check below link for detail.
  # https://utteranc.es/
  utterances:
    repo:                "joannachou713/joannachou713.github.io"
    issue-term:          "pathname"
    label:               "Comments"
    theme:               "github-light"
  ```
* 接下來可以在 `_includes/my-comment.html` 設定留言如何顯示，主要就是貼上剛剛在 Utterances 官網上設定好的 script，這邊不同的地方只有帶入我在 config.yml 裡設定的參數而已
  {%raw%}
  ```erb
    {% assign provider = site.comments.provider | default:"disqus" %}
    {% if provider == "utterances" %}
      {% assign utterances = site.utterances %}
      {% if utterances.repo %}
        <script src="https://utteranc.es/client.js"
                repo={{ utterances.repo }}
                issue-term={{ utterances.issue-term }}
                label={{ utterances.label }}
                theme={{ utterances.theme }}
                crossorigin= "anonymous"
                async>
        </script>
      {% endif %}
    {% endif %}
  ```
  {%endraw%}

* 最後就是把 comment section 加到 post layout 裡、{%raw%}`{{ content }}`{%endraw%}之後加上下面這三行，讓每篇文章下面都有留言區，這樣就完成了！
  {%raw%}
  ```erb
  {%- if site.comments.provider -%}
    {%- include my-comment.html -%}
  {%- endif -%}
  ```
  {%endraw%}

> 上課不認真都在搞這些有的沒的🤡

<font color="grey" style="font-size: 24px">Reference</font>
* [Utterances official site](https://utteranc.es/)
* [Use Utterances/Giscus for Jekyll Comments System](https://lazyren.github.io/devlog/use-utterances-for-jekyll-comments.html)
