---
layout: post
title: Dockerize Rails App (施工中...)
date: 2022-02-19 22:51 +0800
tags: update deploy docker-compose rails
---
終於補到這篇了！需要 Dockerize 的原因很簡單，因為本地環境被我自己搞爆了😀 簡單來說就是我七八月時開發的專案，隔了四個月之後再回來繼續支援時，發現整個環境出一堆問題（基本上都是 M1 chips 相關的），不管怎麼改環境設定，下完指令之後永遠都會 crash，修到最後我也懶得繼續找原因了，想說剛學過 Docker，其他資深的同事們也有這樣搞過，就想說不然我也來試試看吧：）

這個專案用的 Ruby 版本是 2.6.9（所以才會在 M1 的電腦上充滿問題，最一開始在安裝 2.6.9 版的 Ruby 時就已經動過一些手腳了：在 make binary 出問題的話可以先下 `CFLAGS="-Wno-error=implicit-function-declaration" rvm install x.x.x` 再安裝 by 老闆）還會需要用到 Redis, PostgresQL(DB), Webpacker，就都是一些開發 Rails 時所需最基本的配置，並在容器化後使用 docker-compose 來協調這些配置。

## Docker-compose 簡介


## Docker-compose Services
跟上面提到的配置一樣，再加上需要測試用的環境之後，我們在 docker-compose 裡總共會需要以下五個服務: `redis`, `db`(postgres), `app`, `test`, `webpacker`

### Redis
- Redis settings
    - [ruby on rails - Sidekiq Error connecting to Redis on 127.0.0.1:6379 (Errno::ECONNREFUSED) on docker-compose - Stack Overflow](https://stackoverflow.com/questions/34729752/sidekiq-error-connecting-to-redis-on-127-0-0-16379-errnoeconnrefused-on-doc) （[正確答案]: 要改的是 development 的連結😇）

### DB(PostgresQL)
- `config/database.yml`
    - host: `AppName_db`
    - username: `postgres` -> image default setting
    - password: `password` -> image default setting
    - comment out origin setting of `development.database`
- Reset database: command at app container
- `bundle exec rake db:create`...在開完機器後要記得去下...

### App
#### Dockerfile for Rails App
* Base image: `node:14-alpine`, `ruby:2.6.9-alpine`
* Env variables
* Copy entrypoint scripts and grant execution permission
* Copy everything from node image
* Install all dependencies(not including node)
* Install bundler
* WORKDIR
* EXPOSE
* ENTRYPOINT


### Test
- 測試環境
    - `test` service 在建立完之後本來就會死掉
    - 執行測試：`docker-compose run --rm AppName_test rspec`

### Webpacker
- Webpack settings: `config/webpacker.yml`
    - dev_server:
        - host: `webpacker` (→ webpacker container name)
        - public: `0.0.0.0:3035` (→ `0.0.0.0` 為 rails host)
- Webpack container settings:
    - 使用與 rails app container 相同的 Dockerfile、相同的 volume
    - `Node.js` 設定 v14：
        - `Dockerfile.dev`: 用兩層  base image
            - `FROM node:14-alpine as node`
                - 下載 python2
                - `RUN apk add --update --no-cache python2 && ln -sf python2 /usr/bin/python`
            - `FROM ruby:2.6.9-alpine`
                - 把 node 那邊的設定複製過來：`COPY --from=node . .`
    - 要有 lock file ==
    - Heap out of memory error: [ref](https://blog.m4x.io/2021/webpack-how-to-fix-out-of-memory/)
        - Set environment variable:
        - `- NODE_OPTIONS=--max_old_space_size=4096`

- Sass-loader ?
    - remove @rails/sass-loader (???
    - [Rails 6.1 sass-loader without asset pipeline (webpacker) - Stack Overflow](https://stackoverflow.com/questions/65256563/rails-6-1-sass-loader-without-asset-pipeline-webpacker)
    - 全部刪掉重新安裝就可以了？？？WTF?????

### Sidekiq（尚未加入）
  - [How to put sidekiq into Docker in a rails application? - Stack Overflow](https://stackoverflow.com/questions/33563161/how-to-put-sidekiq-into-docker-in-a-rails-application)

---
## 所有改動過的檔案列表
* `config/cable`: `+   3   url: "redis://AppName_redis:6379/1"`
* `config/database.yml`:
    * host: AppName_db(db service name)
    * username: postgres
    * password: postgres
    * database: postgres
* `config/environments/development.rb`: `+  49   config.action_mailer.default_url_options = { host: '0.0.0.0:3000' }`
* `config/settings.yml`: `~  65     host: localhost:3000`
* `config/webpacker.yml`:
    * `~  57     host: webpacker`
    * `~  59     public: 0.0.0.0:3035`



<font color="grey" style="font-size: 24px">Reference</font>
- Dockerfile base: [Rails 6 development with Docker and Docker Compose](https://betterprogramming.pub/rails-6-development-with-docker-55437314a1ad)
- Webpacker container: [rails開發環境容器化實戰指南](https://medium.com/@joehwang.com/rails%E9%96%8B%E7%99%BC%E7%92%B0%E5%A2%83%E5%AE%B9%E5%99%A8%E5%8C%96-505dba2c9678)
