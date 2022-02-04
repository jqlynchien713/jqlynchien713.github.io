---
layout: post
title:  "部落格開始營業！（還有一些 Jekyll 的設定）"
date:   2022-02-03 16:34:53 +0800
tags: jekyll update
---
今天終於下定決心好好搞這件事了，之後會把學到的重要東西都記錄在這

目前會以實習學到的東西為主，學校那邊等規劃完再說（？

希望我可以持之以恆，不要半途而廢🥲

---

<br>
順便紀錄一下如何架出這個網站，這裡是選用 ruby 的靜態網頁框架 jekyll 搭出來的，基本上我只有照著官方的文件 new 出一個 project 而已，比較麻煩的地方是改變網站的 styling

在查了一堆資料、改了一堆設定之後還是沒結果，最後決定重開一個新的 project，然後在專案的根目錄開一個叫 `assets/` 的資料夾，然後放一個 `main.scss` 的檔案（內容如下），就好了！！（神奇）（這個檔案甚至是從 minima gem 裡複製來的）

```sass
---
# Only the main Sass file needs front matter (the dashes are enough)
---
// 引入的 Google Fonts
@import url('https://fonts.googleapis.com/css2?family=Roboto+Mono:wght@300;400&display=swap');

// Minima 主題設定的參數，要在引入 minima stylesheet 之前就先設定好要 override 的參數
$base-font-family: 'Roboto Mono', monospace;
$base-font-weight: 300;
$brand-color: #46CFC5;
$code-bg: #D5E8ED;
$link-color: #4265DC;

// 想要另外宣告的 styling
.rss-subscribe > a, a.u-email, span.username{
  color: $brand-color;
  &:visited{
    color: darken($brand-color, 15%);
  }
}

// 最後再引入 minima 的 stylesheet
@import "minima";
```

當 jekyll 在 build 時，`<source>/assets/` 的優先度高於 `<theme gem>/assets/`，因此可以在 `<source>/assets/` 指定想要覆蓋掉的 styling；對於其他資料夾（like `_includes`, `_layout` ... etc.）也是一樣道理，所以如果想要改主題的設定的話，只要把想要修改的內容從 gem 複製到專案的根目錄（但架構要跟 gem 裡是一樣的），就可以客製自己要的主題了。

最後就是發部到 Github Pages 上，真的比我想像的還要快超多，只要在 Github 建立一個 `<username>.github.io` 的 repo，然後把整個專案推上去就可以用了！！！（非常神奇）（原本還以為可能又要折騰一兩小時）


<font color="grey" style="font-size: 24px">Reference</font>
* [Themes Configuration](https://jekyllrb.com/docs/themes/)
