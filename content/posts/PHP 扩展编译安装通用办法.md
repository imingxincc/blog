---
title: "PHP 扩展编译安装通用办法"
date: 2020-09-15T09:47:55+08:00
draft: true
tags: ["PHP扩展"]
categories: ["PHP"]
---
今天是2018年4月17日。

今天搭建本地开发环境，刚好要用到 Memcached，于是便尝试自己去编译安装。将过程整理如下：

<!--more-->

1. 去[pecl.php.net](http://pecl.php.net "pecl.php.net")寻找扩展源码并下载解压

```bash
$ sudo wget http://pecl.php.net/get/memcached-x.x.x.tgz
$ sudo tar -zxvf memcached-x.x.x.tgz
```

2. 进入解压目录(/path/memcached)

3. 生成 configure 文件

```bash
$ sudo /xxx/path/php/bin/phpize --with-php-config=/xxx/path/php/bin/php-config
```

4. 运行 configure 文件

```bash
$ sudo ./configure --with-php-config=/xxx/path/php/bin/php-config
```

5. 安装扩展

``` bash
$ sudo make && make install
```

6. 将生成好的 .so 文件路径加入到 php.ini 中

`extension=/xxx/path/memcached.so`

7. 重启 Web 服务器

参考：[PECL 扩展库安装](http://php.net/manual/zh/install.pecl.php "PECL扩展库安装")