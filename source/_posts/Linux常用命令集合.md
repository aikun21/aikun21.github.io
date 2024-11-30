---
title: Linux常用命令集合
date: 2017-10-23 13:59:05
tags:
    - 服务器配置
    - linux
    - 后台
---

# centos

## 添加nginx网站

## 防火墙

关闭虚拟机防火墙：

关闭命令： ` service iptables stop`

永久关闭防火墙：`chkconfig iptables off`

两个命令同时运行，运行完成后查看防火墙关闭状态
`service iptables status`

+ 关闭防火墙-----`service iptables stop `
+ 启动防火墙-----`service iptables start `
+ 重启防火墙-----`service iptables restart` 
+ 查看防火墙状态--`service iptables status `
+ 永久关闭防火墙--`chkconfig iptables off` 
+ 永久关闭后启用--`chkconfig iptables on`
