---
title: "服务器连接mysql空闲时出现close_wait"
date: 2017-09-21T15:52:57+08:00
draft: false
categories: [mysql]
tags: [mysql, server, close_wait]
Summary: 服务器连接mysql空闲时出现close_wait
---

## 问题描述

　　最近在查看我们游戏服务器的日志时发现每过一段时间，驱动都会报如下的错误日志：
```
packets.go:124: write unix @->/var/run/mysqld/mysqld.sock: write: broken pipe
```

## 环境
* mysql驱动: github.com/go-sql-driver/mysql (git SHA: be22b30)
* Go version: go1.9 darwin/amd64
* mysql version: Ver 5.7.19
* Server OS: Ubuntu 17.04

## 过程

　　然后开始查资料，找到了[这个链接](https://github.com/go-sql-driver/mysql/issues/446) 。
大意就是，mysql有个```wait_timeout```变量，如果一个链接在这个这个变量设置的时间段内都是空闲的，那么mysql会主动断开这个连接。但是我们是使用的这个mysql驱动无法移除```database/sql.DB```连接池中已经无效的连接，导致客户端(我们的服务器)这边再次使用这个已经被断开的连接的时候就会报错。

　　查询数据库获得如下结果: 

![图片](../../img/mysql-show-timeout.png)

## 解决办法
　　将MaxLifetime值设置为wait_timeout的一半即可:```db.SetConnMaxLifetime(time.Second * 14400)```

## 又出问题了

　　但是这样只能处理时间正常流逝的情况，为了手动改时间不出这种问题，我们决定每次改时间后将mysql也一同重启。但是由此又引出了新的问题。在我们的服务器重启运行半分钟左右(期间会有数据库查询等操作)，发现日志里面多了```Invalid connection```的错误，使用```netstat -an | grep 3306```查看发现了大量```close_wait```状态的连接。
这种情况显而易见，肯定是mysql将我们服务器的连接断开了，猜测应该和前面的情况一致，是由于连接空闲导致的。但是这并不合理，根据查询结果，我们mysql的```wait_timeout```变量值为28800，也就是说至少得空闲8个小时mysql才会主动断开连接，现在这点时间完全没有那么久。

## 继续查资料、分析

　　接着又是各种查资料，各种分析，最后在mysql的配置文件中发现，我们配置了```wait_timeout=30```。看来问题就出在这里了，空闲30秒后mysql就主动断开了连接，与实际情况相符合。那为什么我们使用```show variables like '%timeout%';```查询的结果是28800呢。原来使用命令查询得到的```wait_timeout```的值取决于```interactive_timeout```。虽然```wait_timeout```实际上是生效的，但是我们看到的，都是```interactive_timeout```的值。

## 更多
　　根据[这个链接](http://blog.phpdr.net/mysql-%E4%BA%A4%E4%BA%92%E6%A8%A1%E5%BC%8F%E5%92%8C%E9%9D%9E%E4%BA%A4%E4%BA%92%E6%A8%A1%E5%BC%8F.html)的描述，使用终端登录mysql进行连接时，mysql空闲断开取决于```interactive_timeout```，而我们使用非终端的形式连接时，mysql空闲断开取决于```wait_timeout```。

