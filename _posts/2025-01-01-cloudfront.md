---
layout: post
title: cloudfront forward host header
date: 2025-01-01 00:00 +0800
tags: update CloudFront
---

- 現況分析
    - 在 distribution 為 custom origin 時，如果沒有設定 forward host header，則有可能會收到 502 的結果
    - 不 forward host header 時，cloudfront 會自動將 host header 填入 origin server 的 FQDN
        
        https://arc.net/l/quote/jxzmarwb
        
    - 在 virtual hosting 的架構下，server 或 reverse proxy 會需要依照 host header，去找現在這個 request 應該要去哪個服務 (virtual host)
    - 當 server 或 proxy 找不到 request host header 所指向的 host 時：
        - server 會回傳
            1. 404 not found：表 server 沒辦法處理該 host 的內容（default virtual host 的內容）
            2. 400 bad request：表 host header 無效、不存在
        - proxy 會回傳 502，代表他不知道這個 request 應該要去哪個 host
- 需要 forward  host header 的原因：origin server 的 host name 設定跟 origin domain name 不同
- 相關議題
    - virtual hosting：讓 server 和 machine 脫鉤，且可以獨立處理不同 server 請求的技術（by domain name & host name）
    - host header 用途：在使用 virtual hosting 的架構時，告訴機器現在這個 request 要送到哪個 host
    - host vs domain → 店面 / 地址
    - 觸發 502 的原因
