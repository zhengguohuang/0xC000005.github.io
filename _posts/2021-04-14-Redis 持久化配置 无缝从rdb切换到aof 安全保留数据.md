---
layout:     post
title:      Redis 持久化配置 无缝从rdb切换到aof 安全保留数据
subtitle:   
date:       2021-04-14
author:     zhengguohuang
header-img: img/post-bg-rixi2.jpg
catalog: true
tags:
    - Redis
---

## 问题

redis默认持久化配置rdb，但是如果贸然切换配置到aof方式，重启会导致数据丢失

## 根本原因

- rdb方式默认将数据持久化存储到dump.rdb文件下
- aof方式默认将数据写操作记录到appendonly.aof文件下
- 如果同时开启两种方式，重启会默认加载aof文件
- Redis默认只开启rdb
- 综上，如果默认是rdb方式，然后贸然切换到aof，重启会读取aof文件，但这个时候aof文件是空的，则会导致Redis被清空

![image-20210401190043941](https://gitee.com/zhengguohuang/img/raw/master/img/image-20210401190043941.png)

## 解决方法

在redis控制台动态配置打开aof方式，在shutdown安全退出后，自动记录了当前所有记录到aof文件，再修改redis文件配置打开aof方式，启动redis时会自动加载之前安全退出保存的aof数据

1. 进入Redis

   ```
   ./redis-cli
   ```

2. 动态修改配置并退出

   ```
   save  # 收到触发rdb存储数据
   CONFIG SET appendonly yes  # 动态配置
   save
   shutdown save # 安全退出并存储数据
   ```

3. 修改Redis配置，打开aof

   ```
   appendonly yes
   ```

4. 启动Redis

   ```
   service redis start
   ```

   