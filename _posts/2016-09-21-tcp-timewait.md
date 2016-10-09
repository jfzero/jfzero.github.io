---
layout: blog
categories: tcp
title: tcp连接出现大量TIME_WAIT解决方法
tags: tcp
excerpt: tcp连接出现大量TIME_WAIT解决方法
---

使用ab进行性能测试的时候，发现tcp连接中出现大量的TIME\_WAIT，原因是由于tcp进行四次握手后时TIME\_WAIT状态还需要等2MSL后才能返回到CLOSED状态。
网上查了一下，可通过修改内核参数加以解决，编辑文件/etc/sysctl.conf，加入以下内容：

```
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
```

各个参数的含义如下：

* `net.ipv4.tcp_syncookies = 1`表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭。
* `net.ipv4.tcp_tw_reuse = 1`表示开启重用。允许将TIME\_WAIT sockets重新用于新的TCP连接，默认为0，表示关闭。
* `net.ipv4.tcp_tw_recycle = 1`表示开启TCP连接中TIME\_WAIT sockets的快速回收，默认为0，表示关闭。
* `net.ipv4.tcp_fin_timeout`用以修改系默认的TIMEOUT时间。

然后执行`/sbin/sysctl -p`让参数生效。

其他参数说明如下：

* `net.ipv4.tcp_keepalive_time = 1200`表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为20分钟。
* `net.ipv4.ip_local_port_range = 1024 65000`表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。
* `net.ipv4.tcp_max_syn_backlog = 8192`表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。
* `net.ipv4.tcp_max_tw_buckets = 5000`表示系统同时保持TIME\_WAIT套接字的最大数量，如果超过这个数字，TIME\_WAIT套接字将立刻被清除并打印警告信息。默认为180000，改为5000。对于Apache、Nginx等服务器，上几行的参数可以很好地减少TIME\_WAIT套接字数量，但是对于Squid，效果却不大。此项参数可以控制TIME\_WAIT套接字的最大数量，避免Squid服务器被大量的TIME\_WAIT套接字拖死。
