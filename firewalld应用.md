---
title: firewalld应用
date: 2016-12-21 01:04:34
tags:
- firewalld
category:
- 服务器
- 工具
- 防火墙
---

以下是平时使用的时候用到的一些关于firewalld相关的一些操作，作为笔记，
留待后期使用参考。

<!-- more -->

1. 放行nginx http服务
    1.1 查询firewalld支持的服务
>>> shell
firewadll-cmd --get-services
>>>

    1.2 使用设置当前区域放行http服务
>>> shell
firewall-cmd --zone=public --add-service=http --permanent 
>>>

    1.3 重启firewalld服务
>>> shell
systemctl restart firewalld
>>>
