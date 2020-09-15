---
title: "Supervisor 使用教程"
date: 2020-09-15T18:21:56+08:00
draft: true
tags: ["进程管理"]
categories: ["工具"]
---

### 简介
[Supervisor](http://supervisord.org/) 是一个由 Python 开发的 Linux/Unix 系统上的进程管理工具。它可以将一个普通的命令以守护进程的方式运行并监控制进程状态，当进程异常退出时能够自动重启。

<!--more-->

### 安装
```bash
pip install supervisor
```

### 生成配置文件
```bash
echo_supervisord_conf > /etc/supervisord.conf
```

### 修改配置文件
```bash
[include]
files = etc/supervisord.d/*.ini # 包含其他的配置文件
```

## 配置自己的 Program
```bash
mkdir /etc/supervisord.d/ # 根据配置文件中其他配置文件加载项的配置创建对应的配置文件目录
```

## 下面以 websocket 服务为例
```bash
cd /etc/supervisord.d/
touch super-websocket.ini
```

### super-websocket.ini 详细内容如下：
```bash
[program:websocket] ; 进程名称，不要重复，是 program 的唯一标识
directory = /data/wwwroot/projec/ ; 程序的启动目录
command=artisan websocket:start ; 运行目录下执行命令，启动 websocket 服务
stdout_logfile_maxbytes = 100MB ; stdout 日志文件大小，默认 50MB
stdout_logfile_backups = 10 ; stdout 日志文件备份数
stdout_logfile=/data/wwwroot/projec/storage/logs/supervisor-websocket.log ; stdout 日志文件(需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录)
redirect_stderr=true ; 把 stderr 重定向到 stdout，默认 false
autostart=true ; 当 supervisor 启动时，程序将会自动启动
autorestart=true ; 程序异常退出后自动重启
numprocs=1 ; 进程数
user=www ; 运行程序的用户
startsecs=3 ; 程序重启时候停留在 runing 状态的秒数 (启动 3 秒后没有异常退出，就当作已经正常启动了)
startretries=100000 ; 启动失败时的最多重试次数
```
### 启动
```bash
supervisord -c /etc/supervisord.conf
```
### 查看进程
```bash
ps -aux | grep supervisord
```

### 重启
```bash
supervisorctl -c /etc/supervisord.conf reload # 重启
supervisorctl -c /etc/supervisord.conf shutdown # 注意这里将 supervisord 进程关闭，但通过 supervisord 启动的进程没有关闭
```

### 常用命令
```bash
supervisorctl status # 查看当前进程状态
supervisorctl stop  websocket # 停止一个名为 websocket 的进程
supervisorctl start websocket # 启动一个名为 websocket 的进程
supervisorctl restart websocket # 重启指定进程
supervisorctl restart all # 重启全部进程
supervisorctl reread # 重新加载配置文件，不重启
supervisorctl update # 重新加载配置文件，并将受影响的程序重启
```
说明：可以直接在系统shell中执行，也可以先执行 supervisorctl，进入 supervisorctl_shell 中执行相应的命令。

### Centos 下设置 supervisord 开机自启
```bash
[Unit]
Description=Supervisor Daemon

[Service]
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf
ExecStop=/usr/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/bin/supervisorctl $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

### 启动服务
```bash
systemctl enable supervisord
```

### 验证一下是否为开机自启
```bash
systemctl is-enabled supervisord 
```

### 参考：
[Supervisord管理进程实践](https://thief.one/2018/06/01/1/)  
[Python supervisor 强大的进程管理工具](https://juejin.im/entry/5cbde1335188250a7f630c2a)  
[Supervisor进程管理&开机自启](https://www.jianshu.com/p/03619bf7d7f5)
