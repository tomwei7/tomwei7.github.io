---
layout: post
title:  "9102 年在国内如何快速的下载 Docker 镜像, 现存 Docker 镜像源横评"
date:   2019-07-12 20:57:00 +0800
excerpt_separator: <!--more-->
categories: docker
---

Docker 从 2013 发布一直广受瞩目，但是随着 k8s 等项目的发展，Docker Swarm 的失败，Docker 作为一个底层的容器运行时越来越边缘化，而且随着例如 [rkt](github.com/rkt/rkt), [pouch](https://github.com/alibaba/pouch) 等项目的出现，Docker 作为底层容器运行时的地位也在受到挑战。

但是就目前来说 Docker 仍然是开发者快速部署开发依赖的最好选择，而且但是即使 Docker 不在了，Docker Image 也将作为一个标准的容器镜像格式保留下来。但是 2018 年低 Docker 却悄悄关掉了 Docker CN 中国的镜像源，众所周知国内的网络非常复杂，访问国外网站极其缓慢。所以这里对目前现存的几个 Docker 镜像源进行横评，为大家选择镜像源提供参考。

### 主要对以下五个镜像源进行测试

[Azure 中国镜像  `https://dockerhub.azk8s.cn`](https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md#22-container-registry-proxy)

[阿里云加速器  (需要登录阿里云账户)](https://cr.console.aliyun.com/cn-hangzhou/mirrors)

[七牛云加速器 `https://reg-mirror.qiniu.com`](https://kirk-enterprise.github.io/hub-docs/#/user-guide/mirror)

[中科大镜像源](https://mirrors.ustc.edu.cn/help/dockerhub.html)

[DaoCloud 加速器](https://www.daocloud.io/mirror)


<!--more-->

### 测试主要包括两项

1、已缓存镜像速度: 通过拉取 Docker Hub 较为热门的 `couchbase` 镜像测试镜像源已缓存镜像的下载速度

2、回源镜像速度: 通过拉取我本人自制的放在 Docker Hub 上的镜像，理论上应该不会被任何镜像缓存，测试回源速度

### 测试结果

已缓存的镜像速度 测试使用 `postgres:latest` 镜像，大小约 117 MB

|      | Azure 中国 | 阿里云加速器 | 七牛云加速器 | 中科大   | DaoCloud 加速器 |
| ---- | ---------- | ------------ | ------------ | -------- | --------------- |
| 耗时 | 12.446s    | 13.324s      | 7:10.92s     | 3:36.48s | 18.576s         |

回源镜像速度 测试使用个人打包的一个镜像大小约 268 MB

|      | Azure 中国 | 阿里云加速器 | 七牛云加速器 | 中科大   | DaoCloud 加速器 |
| ---- | ---------- | ------------ | ------------ | -------- | --------------- |
| 耗时 | 44.696s    | 25.356s      | 1:09.56s     | 9:20.40s | 38.069s         |

### 结论

镜像源速度 阿里云 > Azure 中国 > DaoCloud > 中科大 > 七牛云加速器

由于阿里云加速器需要注册阿里云账户开启容器服务之后才有，所以这里强烈推荐使用 [Azure 中国镜像  `https://dockerhub.azk8s.cn`](https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md#22-container-registry-proxy) 的镜像加速服务。

这次测试比较意外的是七牛云的加速器竟然速度这么慢，感觉已经处于无人维护的状态了，在拉取 postgres 镜像过程中多次失败。强烈不推荐使用
