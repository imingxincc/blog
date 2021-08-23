---
title: "Tcpdump 入门"
date: 2021-08-23T14:28:24+08:00
draft: true
tags: ["网络抓包", "tcpdump 抓包教程", "linux 抓包"]
categories: ["工具"]
---

今天使用 tcpdump 定位了一些线上接口超时问题，觉得这个东西真的好用。简单记录一些使用方法，方便自己以后参考。毕竟我们也不是天天使用它，时间久了难免也会忘记。

## 命令行格式
tcpdump [ -adeflnNOpqStvx ] [ -c 数量 ] [ -F 文件名 ][ -i 网络接口 ] [ -r 文件名] [ -s snaplen ][ -T 类型 ] [ -w 文件名 ] [表达式 ]

<!--more-->

## 常用参数

* -i 选择要捕获的接口，通常是以太网或无线网卡，可以是 vlan 或其他特殊接口。如果系统只有一个网络接口，则无需指定
* -l 使标准输出变为缓冲形式
* -nn 单个 n 表示不解析域名，直接显示 IP 地址；两个 n 表示不解析域名和端口
* -c 在收到指定数量的包后，tcpdump 就会停止
* -w 直接将包写入文件中，并不分析和打印出来（之后可以将抓包文件下载到本地，使用 Wireshark 进行分析）
* -s 指定记录 package 的大小，常见-s 0，代表最大值 65535
* -X 直接输出 package data 数据，默认不设置，只能通过 -w 指定文件进行输出
* -p 不让网络进入混杂模式
* -e 显示链路层信息，tcpdump 默认情况下不会显示链路层信息，使用 -e 选项可以显示源和目的 MAC 地址，以及 VLAN tag 信息
* -A 表示使用 ASCII 字符串打印报文的全部数据，这样可以使读取更加简单，方便使用 grep 等工具解析输出内容
* -v 使用 -v, -vv, -vvv 来显示更多的详细信息

## 常用表达式

* 关于类型的关键词：
1. host
2. net
3. port

* 传输方向的关键词
1. src
2. dst
3. dst or src
4. dst and src

* 协议的关键词
1. fddi
2. ip
3. rarp
4. tcp
5. udp

* 逻辑预算
1. 'not', '!' 非运算
2. 'and', '&&' 与运算
3. 'or', '||' 或运算

* 其他重要的关键词
1. gateway
2. broadcast
3. less
4. greater

## 常用示例

1. http 数据包抓取（直接在终端输出 package data）

```bash
tcpdump tcp port 80 -n -X -s 0 # 指定 80 端口进行输出
```

2. http 数据包抓取（向指定文件输出 package data）

```bash
tcpdump tcp port 80 -n -s 0 -w /tmp/tcp.cap # 可以将数据包文件下载到本地后使用 Wireshark 进行分析
```

3. 结合管道流

```bash
tcpdump tcp port 80 -n -s 0 -X | grep xxx
```

4. mod_proxy 反向代理抓包

线上服务器我们使用 nginx 做了反向代理。（80 ==> 8080）

```bash
# 抓取 80 端口的数据：
tcpdump -i eth0 tcp port 80 -n -s 0 -w /tmp/eth0_80.cap

# 抓取 8080 端口的数据：
tcpdump -i lo tcp port 8080 -n -s 0 -w /tmp/lo_8080.cap
```

5. 只监控特定 IP 的主机

```bash
tcpdump tcp host 10.0.2.2 and port 8083 -s 0 -X # 这里 host 指示只监听该 IP
```

6. 只抓取 HTTP GET 请求

```bash
# 0x474554 对应 ASCII 码的 GET
# 0x20 对应 ASCII 码的空格
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

7. 只抓取 HTTP POST 请求

```bash
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
```

## 拓展阅读

### 通常情况下一个正常的 TCP 链接都会有以下三个阶段：

1. TCP 三次握手建立连接
2. 数据传送
3. TCP 四次挥手断开连接

### 相关概念说明：

* SYN 同步序列编号，Synchronize Sequence Numbers
* ACK 确认编号，Acknowledgement Number
* FIN 结束标志，FINish

### TCP 三次握手（创建 open）

* 客户端发起一个和服务端创建 TCP 连接的请求，这里是 SYN(j)
* 服务端接受到客户端的创建请求后，返回两个信息：SYN(k) + ACK(j+1)
* 客户端再接收到服务端的 ACK 信息校验成功后(j与j+1)，返回一个信息：ACK(k+1)
* 服务端这时接收到客户端的 ACK 信息校验成功后(k与k+1)，不再返回信息，后面就进入数据通讯阶段

### 数据通讯

* 客户端/服务端 read/write 数据包

### TCP 四次挥手（关闭 finish）

* 客户端发起关闭请求，发送一个信息：FIN(m)
* 服务端接受到信息后，首先返回 ACK(m+1)，表明自己已经收到消息
* 服务端再准备好关闭之前，最后发送给客户端一个 FIN(n) 消息，询问客户端是否准备好关闭了
* 客户端接受到服务端发送的消息后，返回一个确认信息：ACK(n+1)
* 最后，服务端和客户端在双方都得到确认时，各自关闭或者回收对应的 TCP 连接

### 详细的状态说明及 Linux 相关参数

```bash
sysctl -a # 列出系统内核参数
```

1. SYN_SEND
    * 客户端尝试链接服务端，通过 open 方法。也就是 TCP 三次握手中的第一步之后，注意客户端状态
    * sysctl -w net.ipv4.tcp_syn_retries = 6，做为客户端可以设置 SYN 包的重试次数，默认 5 次

2. SYN_RECEIVED
    * 服务端接受创建请求的 SYN 后，也就是 TCP 三次握手的第二步，发送 ACK 数据包之前
    * sysctl -w net.ipv4.tcp_max_syn_backlog = 1024，设置该状态的等待队列数，默认 1024，调大后可以防止 SYN_FLOOD 攻击
    * sysctl -w net.ipv4.tcp_syncookies = 1，打开 syncookie，在 syn backlog 队列不足的时候，提供一种机制临时将 syn 链接换出
    * sysctl -w net.ipv4.tcp_synack_retries = 2，做为服务端返回 ACK 包的重试次数，默认 5 次

3. ESTABLISHED
    * 客户端接收到服务端的 ACK 包后的状态，服务端在发出 ACK 在一定时间后即为 ESTABLISHED
    * sysctl -w net.ipv4.tcp_keepalive_time = 7200，默认为 7200 秒（2小时），系统针对空闲连接会进行心跳检查，如果超过 net.ipv4.tcp_keepalive_probes * net.ipv4.tcp_keepalive_intvl = 默认 11 分钟，终止对应的 TCP 连接，可适当调整心跳检查频率

4. FIN_WAIT1
    * 主动关闭的一方，在发出 FIN 请求后，也就是在 TCP 四次挥手的第一步

5. CLOSE_WAIT
    * 被动关闭的一方，在接受到客户端的 FIN 后，也就是 TCP 四次挥手的第二步

6. FIN_WAIT2
    * 主动关闭的一方，在接收到被动关闭一方的 ACK 后，也就是 TCP 四次挥手的第二步
    * sysctl -w net.ipv4.tcp_fin_timeout = 60，可以设定被动关闭方返回 FIN 后的超时时间，有效回收连接，避免 SYN_FLOOD

7. LASK_ACK
    * 被动关闭的一方，在发送 ACK 一段时间后（确保客户端已收到），再发起一个 FIN 请求，也就是 TCP 四次挥手的第三步

8. TIME_WAIT
    * 主动关闭的一方，在收到被动关闭的 FIN 包后，发送 ACK，也就是 TCP 四次挥手的第四步
    * sysctl -w net.ipv4.tcp_tw_recycle = 1，打开快速回收 TIME_WAIT
    * sysctl -w net.ipv4.tcp_tw_reuse = 2，快速回收并重用 TIME_WAIT 的连接，貌似和 tw_recycle 有冲突，不能重用就回收
    * sysctl -w net.ipv4.tcp_max_tw_buckets = 5000，处于 TIME_WAIT 状态的最多连接数

### 相关说明

* 主动关闭方在接收到被动关闭方的 FIN 请求后，发送成功给对方一个 ACK 后，将自己的状态由 FIN_WAIT2 修改为 TIME_WAIT，而必须等待2倍的MSL（Maximum Segment Lifetime，MSL是一个数据报在 internetwork 中能存在的时间）时间之后双方才能把状态都改为 CLOSED 以关闭连接。目前 RHEL 里保持 TIME_WAIT 状态的时间为 60 秒。

* keepAlive 策略可以有效的避免进行三次握手和四次挥手的动作

## 参考

[linux tcpdump抓取HTTP包的详细解释](https://blog.51cto.com/u_12295205/3153163)

[Tcpdump 示例教程](https://fuckcloudnative.io/posts/tcpdump-examples/)
