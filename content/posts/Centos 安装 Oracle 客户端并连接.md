---
title: "Centos 安装 Oracle 客户端并连接"
date: 2021-11-03T09:55:47+08:00
draft: true
tags: ["Oracle 客户端安装", "Oracle 连接"]
categories: ["数据库"]
---

最近一直在帮公司搞商品库存实时同步的问题，因为线下使用的是本地机房并且用的是 Oracle 数据库。我们平时查询只能通过接口，而且不能批量操作，很是麻烦。于是我决定自己装一个客户端工具（sqlplus），可以直接连接 Oracle 数据库并查询。下面是一些自己踩过的坑，记录一下：

<!--more-->

开始之前我先简单介绍下我现在的系统环境：

```bash
[vagrant@localhost default]$ cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```

## 安装客户端工具（sqlplus）

官方下载地址：[Oracle Instant Client Downloads for Linux x86-64 (64-bit)](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html)

```bash
# 进入官方下载页，下载并安装 Basic Package 基础工具包和 SQL*Plus Package 工具包
# 下载 Basic Package 安装包
[vagrant@localhost default]$ wget https://download.oracle.com/otn_software/linux/instantclient/213000/oracle-instantclient-basic-21.3.0.0.0-1.el8.x86_64.rpm

# 安装 Basic Package
[vagrant@localhost default]$ rpm -hvi oracle-instantclient-basic-21.3.0.0.0-1.el8.x86_64.rpm

# 下载 SQL*Plus Package 安装包
[vagrant@localhost default]$ wget https://download.oracle.com/otn_software/linux/instantclient/213000/oracle-instantclient-sqlplus-21.3.0.0.0-1.el8.x86_64.rpm

# 安装 SQL*Plus Package
[vagrant@localhost default]$ rpm -hvi oracle-instantclient-sqlplus-21.3.0.0.0-1.el8.x86_64.rpm
```

经过上面几步，Oracle 客户端工具就安装成功了，但是目前还不能立即使用。我们还需要配置相应的环境变量：

```bash
# 安装完成后直接使用 sqlplus 命令会报下面的错误：
# 这是因为我们的 bash 环境变量中缺少库路径的配置项（LD_LIBRARY_PATH）
[vagrant@localhost default]$ sqlplus
sqlplus: error while loading shared libraries: libsqlplus.so: cannot open shared object file: No such file or directory

# 打开用户家目录下的 .bash_profile
[vagrant@localhost default]$ vim ~/.bash_profile
# 添加如下内容到 .bash_profile
# export LD_LIBRARY_PATH=/usr/lib/oracle/21/client64/lib
# export PATH=$PATH:$LD_LIBRARY_PATH

# 使配置立即生效
[vagrant@localhost default]$ source ~/.bash_profile

# 再次执行 sqlplus 命令时会显示如下内容：
[vagrant@localhost default]$ sqlplus

SQL*Plus: Release 21.0.0.0.0 - Production on Mon Nov 1 16:53:25 2021
Version 21.3.0.0.0

Copyright (c) 1982, 2021, Oracle.  All rights reserved.

Enter user-name: 
```

到此我们的客户端软件就算安装完成了，下面我们重点说说它是怎么连接到 Oracle 数据库的。

## 使用 sqlplus 连接 Oracle 数据库

```bash
# 方式一
[vagrant@localhost default]$ sqlplus /nolog

SQL*Plus: Release 21.0.0.0.0 - Production on Mon Nov 1 17:37:49 2021
Version 21.3.0.0.0

Copyright (c) 1982, 2021, Oracle.  All rights reserved.

SQL> connect username@connect_identifier

# 方式二
# 操作系统认证，不需要数据库服务器启动 listener，也不需要数据库服务器处于可用状态。比如我们想要启动数据库就可以用这种方式进入 sqlplus，然后通过 startup 命令来启动。
[vagrant@localhost default]$ sqlplus / as sysdba

# 方式三
# 连接本机数据库，不需要数据库服务器的 listener 进程，但是由于需要用户名密码的认证，因此需要数据库服务器需处于可用状态才行。
[vagrant@localhost default]$ sqlplus username/password

# 方式四
# 连接远程数据库
# sqlplus username/password@ip[:port]/service_name [as sysdba]
[vagrant@localhost default]$ sqlplus username/password@192.168.33.10:1521/service_name

SQL*Plus: Release 21.0.0.0.0 - Production on Mon Nov 1 17:42:00 2021
Version 21.3.0.0.0

Copyright (c) 1982, 2021, Oracle.  All rights reserved.

SQL> 
```