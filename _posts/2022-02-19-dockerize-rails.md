---
layout: post
title: Dockerize Rails App
date: 2022-02-19 22:51 +0800
tags: update deploy docker-compose rails
---
終於補到這篇了！需要 Dockerize 的原因很簡單，因為本地環境被我自己搞爆了😀 簡單來說就是我七八月時開發的專案，隔了四個月之後再回來繼續支援時，發現整個環境出一堆問題（基本上都是 M1 chips 相關的），不管怎麼改環境設定，下完指令之後永遠都會 crash，修到最後我也懶得繼續找原因了，想說剛學過 Docker，其他資深的同事們也有這樣搞過，就想說不然我也來試試看吧：）

這個專案用的 Ruby 版本是 2.6.9（所以才會在 M1 的電腦上充滿問題，最一開始在安裝 2.6.9 版的 Ruby 時就已經動過一些手腳了：在 make binary 出問題的話可以先下 `CFLAGS="-Wno-error=implicit-function-declaration" rvm install x.x.x` 再安裝 by 老闆）還會需要用到 Redis, PostgresQL(DB), Webpacker，就都是一些開發 Rails 時所需最基本的配置，並在容器化後使用 docker-compose 來協調這些配置。

## Docker-compose 簡介
根據官網的介紹，docker-compose 是一個用於管理、執行多容器 Docker Application 的工具。使用 docker-compose 時需要用 YAML 檔來定義 application 的各個 services，接下來即可透過一個指令、從配置中創建並啟動所有服務。這樣做的好處是可以同時定義、協調所有服務，可以避免在個別開啟容器時還要做額外的設定，提高開發的效率。

基本上在設定 docker-compose 時，需要先有各個 service container 要用的 image，不論是自己寫的 Dockerfile，或者是從 docker hub 上 pull 下來的 image 都可以。接下來就是 `docker-compose.yml` 的撰寫，需要先對整個應用程式做全域的設定（如使用的 docker-compose 版本、要有哪些 networks 等），接著再對個別的 services 做細節設定（選用哪裡的 image、port mapping、專案的環境變數...等）

## Docker-compose Configurations
在 `docker-compose.yml` 的開頭需要先對整個 application 做最基本的設定，但因為這邊只是開發環境而已，因此就只有簡單做 Networks & Volumes 的設定而已，主要就是開兩個 Networks(`development` & `test`) 與四個 Volumes(`db_data`, `gem_cache`, `shared_data` & `packs`)。

### Basic Setups
* <ins>Version</ins>: 指定要使用的 docker-compose 版本，目前看到大多數的教學都是用第三版
* <ins>Networks</ins>: 在建起 application 之後，docker-compose 會把所有容器都丟到一個 default network 中，而在同個 network 裡的容器都互相 reachable。如果不想使用預設的 default network 的話也能自己另外宣告，並創造更複雜的 network topology。基本上可以透過這樣的設定達到區隔環境的效果，這也是為什麼這個專案的 networks 設置要分成 development 與 test 的原因
* <ins>Volumes</ins>: 統一宣告所有 images 會用到的 volumes，因為有些 services 可能會需要共用 volumes(像是這個專案裡的 `shared_data` volume 就是拿來讓所有 containers 共用的 volume)。主要的功用是在開發的時候可以同步將本地端修改的內容，mapping 到容器裡的專案，在開發上比較節省時間。
    > Mapping 的格式： `外界:容器`

## Services Configurations
跟上面提到的配置一樣，再加上需要測試用的環境之後，我們在 docker-compose 裡總共會需要以下五個服務: `redis`, `db`(postgres), `app`, `test`, `webpacker`

### Redis
- Docker-compose service 設定
  - <ins>image</ins>: 可以直接選用 docker hub 上的 redis image
  - <ins>command</ins>: 當這個服務開始運行時要執行的指令，aka 執行 redis server 的指令 `redis-server`
  - <ins>networks</ins>: 這邊因為開發與測試環境都會用到 redis，所以兩個 network 都要放進來
  - <ins>volumes</ins>: 這邊只需要 `shared_data`

  ```yml
  AppName_redis:
    image: redis:6.0-alpine
    command: redis-server
    networks:
      - development
      - test
    volumes:
      - shared_data:/var/shared/redis
  ```
- Rails App 設定
  - 需要將 app 內設定的 redis url 改成 docker-compose 裡指定的 service name 格式：`"redis://AppName_redis:6379/1"` (development & test 兩個環境下都需要改) [Reference](https://stackoverflow.com/questions/34729752/sidekiq-error-connecting-to-redis-on-127-0-0-16379-errnoeconnrefused-on-doc)

### DB(PostgresQL)
- Docker-compose service 設定
  - <ins>image</ins>: 這邊也是直接選用 docker hub 上的 postgres container
  - <ins>volumes</ins>: 除了所有 service 共用的 `shared_data` 外，還需要指定一個用來儲存資料庫資訊的 volume(`db_data`)
  - <ins>networks</ins>: 這邊也是兩個環境都會需要 db，因此兩個 network 都需要放
  - <ins>environment</ins>: 用來設定這個 container 的環境變數，這邊需要設定的是資料庫的帳號密碼，而官方 image 預設的帳號為 `postgres` 密碼為 `password`；因為這邊只是測試環境，所以用預設的沒關係，但如果上 staging 或 production 這種正式的環境時，就需要另外用 CI 工具做設定。
  - <ins>ports</ins>: 做 port mapping 用，讓資料庫可以被外界讀取

  ```yml
  AppName_db:
    image: postgres:14-alpine
    container_name: AppName_db
    volumes:
      - db_data:/var/lib/postgresql/data
      - shared_data:/var/shared
    networks:
      - development
      - test
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - 5432:5432
  ```
- Rails App 設定：`config/database.yml`
  - 需要將整個 app 要連結的 db 改成 docker-compose 上的 service，並且設定帳號密碼
  - host: `AppName_db`
  - username: `postgres` -> image default setting
  - password: `password` -> image default setting
- 開啟整個服務後要記得先下 `bundle exec rake db:create` 來創建會用到的 db，否則資料庫的內容會長在一個很奇怪的位置，後續如果要用 rake 的指令操作資料庫會沒辦法讀取到正確的資料

### App
* Dockerfile for Rails App
  * 2 base images `node:14-alpine`, `ruby:2.6.9-alpine`: 這邊會用到兩層 base image 是因為專案裡的 node 版本是 14，不過如果只用 `apk add` 去下載 node 的話都會載到最新的 16 版，因此最後決定載兩層 base image，並且把帶 14 版的設定複製到 ruby image 裡，node 的部分就能正常運作了
  * Env variables
  * Copy entrypoint scripts and grant execution permission
  * Copy everything from node image
  * Install all dependencies(not including node)
    * 這邊要避開下載 node，以免把 14 版的設定覆蓋掉

  ```dockerfile
  FROM node:14-alpine as node
  RUN apk add --update --no-cache python2 && ln -sf python2 /usr/bin/python
  FROM ruby:2.6.9-alpine

  ENV APP_PATH /var/app
  ENV BUNDLE_VERSION 2.2.33
  ENV BUNDLE_PATH /usr/local/bundle/gems
  ENV TMP_PATH /tmp/
  ENV RAILS_LOG_TO_STDOUT true
  ENV RAILS_PORT 3000

  # copy entrypoint scripts and grant execution permissions
  COPY ./dev-docker-entrypoint.sh /usr/local/bin/dev-entrypoint.sh
  COPY ./test-docker-entrypoint.sh /usr/local/bin/test-entrypoint.sh
  COPY --from=node . .
  RUN chmod +x /usr/local/bin/dev-entrypoint.sh && chmod +x /usr/local/bin/test-entrypoint.sh

  # install dependencies for application
  RUN apk -U add --no-cache \
  build-base \
  git \
  postgresql-dev \
  postgresql-client \
  libxml2-dev \
  libidn-dev \
  libxslt-dev \
  yarn \
  imagemagick6 \
  imagemagick6-c++ \
  imagemagick6-dev \
  imagemagick6-libs \
  tzdata \
  less \
  curl \
  bash \
  && rm -rf /var/cache/apk/* \
  && mkdir -p $APP_PATH

  RUN gem install bundler --version "$BUNDLE_VERSION" \
  && rm -rf $GEM_HOME/cache/*

  RUN yarn install --check-file

  # navigate to app directory
  WORKDIR $APP_PATH

  EXPOSE $RAILS_PORT

  ENTRYPOINT [ "bundle", "exec" ]
  ```
* Docker-compose service 設定
  * <ins>image</ins>: 使用上述的 Dockerfile 建出的 image，跟上面幾個服務不一樣，要使用 `build` 來指定建立 image 的資料夾還有 Dockerfile 的位置
  * <ins>volumes</ins>: 除了共用的 `shared_data` 外，還有整個專案的 mapping，需要對應到剛剛 Dockerfile 裡定義的 `$APP_PATH` 上，藉由 Docker bind mounts 達到 hot reloading 的效果；另外還有 `gem_cache` 的 volume，讓 dependencies 可以被清理和重建而不會干擾 app 的其他部分
  * <ins>stdin_open</ins>: 讓容器的標準輸入保持打開
  * <ins>tty</ins>: 將 Docker 分配一個虛擬終端（pseudo-tty）並綁定到容器的標準輸入上，讓我們可以使用 byebug 來進行 debug
  * <ins>entrypoint</ins>: 設定當對 docker-compose 裡的 container 下指令時的進入點
  * <ins>command</ins>: 當啟動容器後要執行的指令
  * <ins>env_file</ins>: 環境設定檔
  * <ins>environment</ins>: 環境變數，可以被寫在環境設定檔裡，這邊會分開寫是因為公司的專案裡已經有預設的環境設定檔了，為了不洗掉原本的設定才另外寫在這個項目下。
  * <ins>depends_on</ins>: 開啟這個服務前需要其他哪些服務的支援

  ```yml
  AppName_app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: AppName_app
    volumes:
      - .:/var/app
      - shared_data:/var/shared
      - gem_cache:/usr/local/bundle/gems
    networks:
      - development
    ports:
      - 3000:3000
    stdin_open: true
    tty: true
    entrypoint: dev-entrypoint.sh
    command:
      - rails server -p 3000 -b 0.0.0.0
    env_file: .env.example
    environment:
      RAILS_ENV: development
      WEBPACKER_DEV_SERVER_HOST: webpacker
    depends_on:
      - AppName_db
      - webpacker
  ```
- ActionMailer 相關設定
  - 如果會用到 ActionMailer 的話需要把 default_url_options 改成 docker-compose 上指定的 ip address，`config/environments/development.rb`: `config.action_mailer.default_url_options = { host: '0.0.0.0:3000' }`

### Test
- Docker-compose service 設定
  - image: 同樣使用 app 的 image
  - volumes: 與 app 的設定相同

```yml
AppName_test:
  image: AppName_AppName_app
  container_name: AppName_test
  volumes:
    - .:/var/app
    - shared_data:/var/shared
    - gem_cache:/usr/local/bundle/gems
  networks:
    - test
  ports:
    - 3001:3000
  stdin_open: true
  tty: true
  entrypoint: test-entrypoint.sh
  command: ["rails", "-v"]
  environment:
    RAILS_ENV: test
    WEBPACKER_DEV_SERVER_HOST: webpacker
  depends_on:
    - AppName_db
    - webpacker
```
- 注意事項
    - `test` service 在建立完之後本來就會死掉
    - 執行測試：`docker-compose run --rm AppName_test rspec`

### Webpacker
最後就是被搞得半死ㄉ webpacker 了！終於要結束了！！！😀🔪
- Rails App's Webpack settings: `config/webpacker.yml`
  - 基本上就是需要把 webpacker 設定成 docker-compose 裡配置的 webpacker 路徑
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
    - 要先執行 `yarn install` 等指令，生成 lock file
    - Heap out of memory error: [ref](https://blog.m4x.io/2021/webpack-how-to-fix-out-of-memory/)
        - Set environment variable:
        - `- NODE_OPTIONS=--max_old_space_size=4096`

```yml
webpacker:
  build:
    context: .
    dockerfile: Dockerfile.dev
  command: ruby bin/webpack-dev-server
  volumes:
    - shared_data:/var/shared
    - gem_cache:/usr/local/bundle/gems
    - .:/var/app
  environment:
    - NODE_OPTIONS=--max_old_space_size=4096
    - NODE_ENV=development
    - RAILS_ENV=development
    - WEBPACKER_DEV_SERVER_HOST=0.0.0.0
  ports:
    - "3035:3035"
  networks:
    - development
```

---
## 所有改動過的檔案列表
* `config/cable.yml`: `url: "redis://AppName_redis:6379/1"`
* `config/database.yml`:
    * host: AppName_db(db service name)
    * username: postgres
    * password: postgres
    * database: postgres
* `config/environments/development.rb`: `config.action_mailer.default_url_options = { host: '0.0.0.0:3000' }`
* `config/settings.yml`: `host: localhost:3000`
* `config/webpacker.yml`:
    * `host: webpacker`
    * `public: 0.0.0.0:3035`
---
好ㄌ....大概就是這樣，寫這篇真的豪累，但是寫 k8s 那兩篇好像會更累🥲？（哭ㄌ



<font color="grey" style="font-size: 24px">Reference</font>
- Dockerfile base: [Rails 6 development with Docker and Docker Compose](https://betterprogramming.pub/rails-6-development-with-docker-55437314a1ad)
- Webpacker container: [rails開發環境容器化實戰指南](https://medium.com/@joehwang.com/rails%E9%96%8B%E7%99%BC%E7%92%B0%E5%A2%83%E5%AE%B9%E5%99%A8%E5%8C%96-505dba2c9678)
