---
layout: post
title: 在 EC2 中開啟 Docker Swarm 群集
date: 2022-06-22 20:38 +0800
tags: update docker docker-swarm COSCUP
---
## Docker Swarm Introduction

- Ref: [(32) Docker Swarm Introduction - Video 28 - YouTube](https://www.youtube.com/watch?v=_riEfxHoUh0&list=PLzZ2RIhyht-PTxy-l5H4iwtCPmNHJu8wD&index=29)

### Structure

![Work.jpg](/assets/img/swarm.png)

### Scalability & Availability

**Differences/Definitions of Scalability & Availability**

- Scalability: 當流量變大/變小時，系統中的 server 數可以橫向擴展或縮減
- Availability: 當部分的機器無法運作時，能夠確保服務不被中斷（自動開啟新的 instance）
- Docker Swarm Features
    1. Scalability：自動在空間足夠的 worker 上開啟新的 container
    2. Availability：以更新 app、降低 downtime 為例
        1. 當需要更新 app 版本時，可以讓 load balancer 自動 route 到新版 app 上
        2. 漸進式取代所有舊版的 app container

### Service vs Task
根據 document 的寫法，service 與 task 的差別如下圖：
![https://docs.docker.com/engine/swarm/images/services-diagram.png](https://docs.docker.com/engine/swarm/images/services-diagram.png)
兩者的區別在於，service 是由 manager 控管的 application，而 task 則是根據 manager 的控制、各個 worker 被指派的任務。

### Stack vs Compose

基本上 stack 可以視為升級版的 compose，並且 stack 的設定檔也可以拿 docker-compose file 來執行（只要是 yml 都行）

## Swarm Mode Implementation

- Ref: [Swarm与Docker的整合——Swarm mode - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/79694462)
- [Getting started with swarm mode, Docker Documentation](https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts) → port prerequistive to set up security group
- Run docker daemon on EC2(amazon linux): [Method 2: Assign Ownership to the Docker Unix Socket](https://phoenixnap.com/kb/cannot-connect-to-the-docker-daemon-error#:~:text=Method%202%3A%20Assign%20Ownership%20to%20the%20Docker%20Unix%20Socket)

### EC2 特殊設定

- SSM53: session manager [參考前一篇文章]({% post_url 2022-06-05-aws-ssh %})
- Security Group Inbounds
    - Allow TCP port 6000-9000 from my IP
    - Allow all TCP inbound from same security group，好處是可以不用特別設定 [docker swarm 指定的傳輸 port](https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts)

### 操作步驟

1. 開三台 EC2，其中一台作為 manager、另外兩台作為 worker
2. 安裝 docker 並執行 docker daemon
    1. 開啟 docker engine：`sudo service docker start`
    2. `sudo chown ec2-user:docker /var/run/docker.sock` （`ec2-user`為 user-name，給予 ec2-user docker engine 的 ownership）

**Manager**

1. 開啟一個 swarm、並將所在的機器指定為 manager：`docker swarm init --advertise-addr [manager-ip]`
    1. 終端機會回傳如何將 worker 加入 swarm 的指令：`docker swarm join --token [token] [manager-ip]:2377`
    2. 如果遺忘加入的指令的話，可以在 manager 下這個指令來找回：`docker swarm join-token worker`

**Worker**

1. 用生成的指令來加入剛剛設定的 swarm：`docker swarm join --token [token] [manager-ip]:2377`

**檢查 swarm 元件**

1. 可以在 manager 下 `docker node ls` 檢查 swarm 內的節點是否符合預期
