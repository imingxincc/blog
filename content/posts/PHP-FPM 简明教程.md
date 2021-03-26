---
title: "PHP-FPM 简明教程"
date: 2021-03-26T09:55:53+08:00
draft: true
target: ["PHP-FPM", "Nginx", "CGI", "FastCGI"]
categories: ["PHP"]
---

作为一名 PHP 程序猿，日常开发中总是免不了与 Nginx、PHP-FPM 打交道。本文整理了一些日常开发工作中的常见问题及一些 PHP-FPM 的简单使用，能力有限，难免会有疏漏或者错误，希望大家批评指正。

<!--more-->

## PHP-FPM 是什么？
FPM (FastCGI Process Manager) 也就是 FastCGI 进程管理器。PHP-FPM 顾名思义就是用于 PHP FastCGI 的进程管理器。

## PHP-FPM、CGI、FastCGI 是什么关系？
* CGI (Common Gateway Interface) 即通用网关接口，它是一种对接应用程序和网络服务器的接口协议，CGI 使外部程序与 Web 服务器之间交互成为可能。CGI 程序运行在独立的进程中，并对每个 Web 请求创建一个进程，在结束时销毁。这种方式非常容易实现，但是效率很差，难以扩展。在高并发情况下，不停的创建和销毁进程使得服务器的开销会变得很大。此外，由于地址空间无法共享，CGI 进程模型限制了资源的重用，如重用数据库连接、内存缓存等等。其实说白了，CGI 就是一种让交互程序与 Web 服务器通信的协议。重点就是协议二字。
* FastCGI (Fast Common Gateway Interface) 快速通用网关接口，是 CGI 协议的增强版。FastCGI 致力于减少 Web 服务器与 CGI 程序之间交互的开销，从而使服务器可以同时处理更多的网页请求。与为每个请求创建一个新的进程不同，FastCGI 使用持续的进程来处理一连串的请求。这些进程由 FastCGI 进程管理器(比如我们本文将要介绍的PHP-FPM)管理。当有一个请求进来时，Web 服务器把环境变量和页面请求通过 Unix 套接字、命名管道或传输控制协议(TCP)传递给 FastCGI 进程，响应通过相同的连接从进程返回到 Web 服务器，然后 Web 服务器再将响应传递给最终的用户。连接可能在响应结束时关闭，但 Web 服务器和 FastCGI 服务进程都将继续，不会被销毁。每个单独的 FastCGI 进程在其生命周期内可以处理许多请求，从而避免了每个请求进程创建和终止的开销。
* FPM (FastCGI Process Manager) 也就是 FastCGI 进程管理器。PHP-FPM 顾名思义就是用于 PHP FastCGI 的进程管理器。

## CGI的工作原理
用户发出请求 -> Web 服务器接受到请求，然后将请求信息通过标准输入(stdin)和环境变量(environment variable)传递给指定的 CGI 程序 -> 启动应用程序进行处理，处理结果通过标准输出(stdout)返回给 Web 服务器 -> Web 服务器拿到处理结果后再通过 HTTP 协议返回给用户
![传统 CGI 的工作原理](/images/1478757541407927.png "传统 CGI 的工作原理")

## FastCGI的工作原理
用户发出请求 -> Web 服务器接收到请求，然后通过 socket 或 tcp 将请求数据转发给 PHP-FPM 进程管理器 -> PHP-FPM 进程管理器拿到请求后交给一个子进程进行处理 -> 子进程处理完成后 PHP-FPM 返回处理结果给 Web 服务器 -> Web 服务器拿到处理结果后再通过 HTTP 协议返回给用户
![FastCGI 的工作原理](/images/1478757572486219.png "FastCGI 的工作原理")

## PHP-FPM 进程管理的几种方式
PHP-FPM 采用 master\worker 的进程管理方式。master 进程管理多个 worker 进程，master 只负责请求转发和进程管理，不负责对具体数据进行处理。 

### static（静态模式）
static - PHP-FPM 启动时子进程的数量是固定的，不会动态增加或减少（pm.max_children: 当 pm 设置为 static 时表示创建子进程的数量，pm 设置为 dynamic 时表示最大可创建的子进程的数量）。设置固定数量的子进程，占用内存高，但在用户请求波动大的时候，对 Linux 操作系统进程的处理上耗费的系统资源低。

### dynamic（动态模式）
dynamic - PHP-FPM 启动时会创建指定数量的子进程（pm.start_servers），当请求量变大时会动态的增加子进程的数量，但不会超过 pm.max_children 所设置的最大值，同样当请求量变小时，也会相应的减少空闲子进程的数量（pm.min_spare_servers，pm.max_spare_servers）。当用户请求波动较大时，进行大量进程的创建与销毁等操作会造成系统负载的波动。但是当请求量小的时候，进程数也会减少，内存占用也会相应的减小。

### ondemand（按需模式）
ondemand - PHP-FPM 启动时不会创建任何子进程，只有当请求时才创建子进程。这种模式一般很少会使用到。

## PHP-FPM 常用配置
* pm：设置进程管理器如何管理子进程。可用值：static，ondemand，dynamic。
* pm.max_children：当 pm 设置为 static 时表示创建的子进程的数量，pm 设置为 dynamic 时表示最大可创建的子进程的数量。
* pm.start_servers：设置启动时创建的子进程数目。仅在 pm 设置为 dynamic 时使用。默认值：min_spare_servers + (max_spare_servers - min_spare_servers) / 2。
* pm.min_spare_servers：设置空闲服务进程的最低数目。仅在 pm 设置为 dynamic 时使用。
* pm.max_spare_servers：设置空闲服务进程的最大数目。仅在 pm 设置为 dynamic 时使用。
* pm.max_requests：设置每个子进程重生之前服务的请求数。对于可能存在内存泄漏的第三方模块来说是非常有用的。如果设置为 '0' 则一直接受请求，等同于 PHP_FCGI_MAX_REQUESTS 环境变量。默认值：0。
* request_terminate_timeout：设置单个请求的超时中止时间。该选项可能会对 php.ini 设置中的 'max_execution_time' 因为某些特殊原因没有中止运行的脚本有用。设置为 '0' 表示 'Off'。可用单位：s（秒），m（分），h（小时）或者 d（天）。默认单位：s（秒）。默认值：0（关闭）。
* request_slowlog_timeout：当一个请求该设置的超时时间后，就会将对应的 PHP 调用堆栈信息完整写入到慢日志中。设置为 '0' 表示 'Off'。可用单位：s（秒），m（分），h（小时）或者 d（天）。默认单位：s（秒）。默认值：0（关闭）。
* slowlog：慢请求的记录日志。默认值：#INSTALL_PREFIX#/log/php-fpm.log.slow。

## 常用操作

获取当前 PHP-FPM 进程数
```bash
ps -ef | grep php-fpm | grep -v grep | wc -l
```

获取当前 PHP-FPM 进程ID
```bash
ps -ef | grep php-fpm | grep -v grep | awk '{print $2}'
```

查看 PHP-FPM 进程树
```bash
# 4924 是我自己服务器上 PHP-FPM master 进程的ID
pstree -p 4924
```

## 性能监控

### top 命令
参数说明：
* -d：指定每两次屏幕信息刷新之间的时间间隔。（也可以通过交互命令 s 进行改变）
* -p：指定进程ID来监控指定进程的状态。
* -i：不显示任何限制或僵尸进程。
* m：切换显示内存信息。
* t：切换显示CPU状态信息。
* c：切换显示命令名称和完整命令行。
* M：根据内存使用大小进行排序。
* P：根据CPU使用率进行排序。
* T：根据时间/累计时间进行排序。
* z：显示颜色

### starce 命令
参数说明：
* -c：统计每一次系统调用所执行的时间、次数和出错的次数等。
* -d：输出 strace 关于标准错误的调试信息。
* -f：跟踪由 fork 调用所产生的子进程。
* -o filename：将进程的跟踪结果输出到相应的 filename 文件中。
* -F：尝试跟踪 vfork 调用。
* -r：打印相对时间关于每一个系统调用。
* -t：在输出中的每一行前面加上时间信息。
* -tt：在输出中的每一行前面加上时间信息，微秒级。
* -ttt：微秒级输出，以秒来表示时间。
* -T：显示每一次调用所耗时间。
* -p pid：追踪指定进程。

使用示例：
```bash
# 利用 nohup 将 strace 转为后台执行，直到 attach 上 PHP-FPM 的进程死掉为止
nohup strace -T -p 13167 > 13167-strace.log &

# 追踪指定进程并进行汇总
strace -cp 4262
```

### 利用 sort\uniq 命令分析汇总 PHP-FPM 慢日志
```bash
# grep -v "^$"：剔除空行
# sort: 对文本进行排序
# uniq -c: 显示唯一的行，并在每行行首加上本行在文件中出现的次数
# sort -k1,1nr: 按照第一个字段，数值排序，且为逆序
# head -10: 取前10行数据
grep -v "^$" /www/var/log/slow.log | cut -d " " -f 3,2 | sort | uniq -c | sort -k1,1nr | head -n 50
```

### sort 命令
参数说明：
* -f：忽略大小写；
* -b：忽略每行前面的空白部分；
* -n：以数值型进行排序，默认使用字符串排序；
* -r：反向排序；
* -u：删除重复行。就是 uniq 命令；
* -t：指定分隔符，默认分隔符是制表符；
* -k [n,m]：按照指定的字段范围排序。从第 n 个字段开始，到第 m 个字（默认到行尾）；

### cut 命令
参数说明：
* -b：以字节为单位进行分割。这些字节位置将忽略多字节字符边界，除非也指定了 -n 标志。
* -c：以字符为单位进行分割。
* -d：自定义分隔符，默认为制表符。
* -f：与 -d 参数一起使用，指定显示哪个区域。
* -n：取消分割多字节字符。仅和 -b 标志一起使用。如果字符的最后一个字节落在由 -b 标志的 List 参数指示的范围之内，该字符将被写出；否则，该字符将被排除。

## 常见问题

### 502 Bad gateway

max_children

如果 max_children 的值设置的较小，那么 PHP-FPM 子进程就会很忙碌，其他请求的等待时间就会变长。如果长时间没有得到处理的请求就会出现 504 Gateway Time-out 这个错误。而如果忙碌中的子进程遇到问题就会出现 502 Bad gateway 这个错误。所以，原则上来讲，这个值设置的越大越好。按每个子进程分配 20MB 内存来算，假设将 max_children 设置成 100，20MB * 100 = 2000MB，也就是说在峰值的时候 PHP-CGI 所消耗的内存在 2000MB 以内。

max_requests

该设置表示一个子进程在处理多少个请求后就会被终止掉。master 进程就会重新创建一个新的子进程。该配置的主要目的是为了避免 PHP 解释器或第三方库造成的内存泄露。502 一般是后端 PHP-FPM 不可用或子进程异常退出等问题造成的。间歇性 502 一般认为是由于 PHP-FPM 进程重启造成的。如果不重启 PHP-FPM 进程，势必会造成内存使用量不断增长。因此 PHP-FPM 作为 PHP-CGI 的管理器，提供了这么一项监控功能。对请求达到指定次数的 PHP-CGI 进程进行重启，保证内存使用量不增长。正式因为这种机制，在高并发中，经常导致间歇性 502 错误。我们可以把该值设置的尽可能大些，需要不断的进行测试来找出一个合适的数值。

request_terminate_timeout

该设置表示一个请求的超时中止时间。该选项可能会对 php.ini 设置中的 'max_execution_time' 因为某些特殊原因没有中止运行的脚本有用。设置为 '0' 表示 'Off'。request_terminate_timeout 和 max_execution_time 这两个选项都是用来配置一个 PHP 脚本的最大执行时间。但是，在 PHP-FPM 中，该参数不会起效。真正能够控制 PHP 脚本最大执行时间的是 php-fpm.conf 配置文件中的 request_terminate_timeout 参数。当超过 request_terminate_timeout 这个时间时，PHP-FPM 不只会终止脚本的执行，还会终止执行脚本的 worker 进程。当 Nginx 发现与自己通信的连接断掉时，就会返回 502 给客户端。

**总结：**

**502 Bad gateway 一般都是由于后端 PHP-FPM 进程不可用或异常退出导致的。**

### 504 Gateway Time-out

504 Gateway Time-out 一般是由 Nginx 等待 PHP-FPM 返回结果超时导致的，Nginx 达到连接超时时间但是后端 PHP-FPM 还没有返回执行结果，Nginx 就会断开连接并返回 504 错误。

```bash
# 重点关注 Nginx 中关于 FastCGI 相关选项的配置：
fastcgi_connect_timeout 300; # 指定 Nginx 与后端 FastCGI Server 连接超时时间
fastcgi_send_timeout 300; # 指定 Nginx 向后端传送请求超时时间
fastcgi_read_timeout 300; # 指定 Nginx 接受后端 FastCGI 响应请求超时时间

# PHP-FPM 中关于处理请求超时时间的配置：
request_terminate_timeout = 100

# 一般情况下，当后者执行时间超过 Nginx 中配置的 FastCGI 最大连接时间时，就会报 504 错误。
```


## 参考资料
* [深入理解 FastCGI 协议以及在 PHP 中的实现](https://mengkang.net/668.html)
* [获取查看PHP-FPM进程相关信息](https://qq52o.me/2636.html)
* [php-fpm 与 Nginx优化总结](https://www.kancloud.cn/digest/php-src/136260)

