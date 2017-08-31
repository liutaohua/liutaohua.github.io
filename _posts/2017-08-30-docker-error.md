---
layout: post
title: docker permission denied错误
categories: docker
description:
keywords: docker, permission denied
---

# [8] System error: permission denied
```
$ docker run -ti hello-world

Error response from daemon: Cannot start container 23e2ee5be7335c1513a792844e8ad558a6df538ccf7ea4bc0a5d013a1bc4ff33: [8] System error: permission denied
```
我使用的是Centos 7,yum安装的Docker version 1.7.1, build 446ad9b/1.7.1,然后进行了升级，下载docker-1.8.3.tgz,替换/usr/bin/docker,之后再去run所有的镜像都是permission denied

解决办法：

1. 关闭防火墙
```
$ system stop firewalld
```
检查依然不能够正常使用
1. 关闭SELinux，临时关闭（不用重启）
```
$ setenforce 0 ##设置SELinux 成为permissive模式
```
能够正常的启动容器，但是我的服务当中需要使用到skydns，虽然容器都能够正常启动，但是域名解析的服务已然不能够使用。
关闭docker的SELinux
```
$ vi /etc/sysconfig/docker  ##删除OPTIONS=’–selinux-enabled’中的–selinux-enabled选项
```
此时已经能够正常的使用服务，但是仅供参考，不建议使用到生产环境当中！！！！