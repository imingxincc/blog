---
title: "PHP 命令行模式的标准输入与输出"
date: 2021-10-15T14:51:48+08:00
draft: true
target: ["PHP 输入输出", "PHP STDIN", "PHP STDOUT", "PHP CLI 输入输出"]
categories: ["PHP"]
---

最近自己一直在学习 C 语言，想多了解一些 PHP 的底层实现和扩展开发。在学习 C 语言的过程中经常会需要使用到标准输入和输出，忽然转念一想如果用 PHP 的话，如何能实现和 C 语言一样的效果呢？下面是自己参考官方手册做了一些总结和示例：

<!--more-->

## PHP 的标准输入与输出

### STDIN 标准输入(CLI专用常量)

```php
<?php
    $line = trim(fgets(STDIN)); // 从 STDIN 读取一行
    fscanf(STDIN, "%d", $number); // 从 STDIN 读取数字
?>
```

```php
<?php
    // 使用 PHP 内置的 URL 风格的封装协议
    $stdin = fopen('php://stdin', 'r');
?>
```

### STDOUT 标准输出(CLI专用常量)

```php
<?php
    $str = "演示标准输出\n";
    fwrite(STDOUT, $str, strlen($str)); // 注意此处使用的是 strlen() 函数，而不是 mb_strlen() 函数。关于它俩的区别，我们后面文章会说。
?>
```

```php
<?php
    // 使用 PHP 内置的 URL 风格的封装协议
    $str = "演示标准输出\n";
    $stdout = fopen('php://stdout', 'w');
    fwrite($stdout, $str, strlen($str));
    fclose($stdout);
?>
```

### STDERR 标准错误输出(CLI专用常量)

```php
<?php
    $str = "演示标准错误输出\n";
    fwrite(STDERR, $str, strlen($str));
?>
```

```php
<?php
    // 使用 PHP 内置的 URL 风格的封装协议
    $str = "演示标准错误输出\n";
    $stdout = fopen('php://stderr', 'w');
    fwrite($stdout, $str, strlen($str));
    fclose($stdout);
?>
```

注：
1. STDIN、STDOUT、STDERR是 CLI 模式下的专用常量，如果在 CGI 模式下使用的话，会报错“使用未定义常量”。(Use of undefined constant STDIN - assumed 'STDIN' (this will throw an Error in a future version of PHP))

2. 在进行输入输出操作时，官方推荐使用 STDIN、STDOUT、STDERR 常量，这样就避免自己去建立诸如 stdin、stdout 的流，也无需自己来关闭这些流。

## 示例演示

```php
<?php
    echo "请输入文字：";
    $str = fgets(STDIN);
    printf("您输入的内容是：%s", $str);
?>
```

读取标准输入后格式化输出：

```php
<?php
    $students = [];
    for ($i = 1; $i < 5; $i++) {
        printf("请输入学生的姓名、年龄和学号：");
        fscanf(STDIN, "%s %d %d", $name, $age, $number);
        $students[$i] = [
            'name' => $name, // 姓名
            'age' => $age, // 年龄
            'number' => $number // 学号
        ];
    }
    printf("序号\t姓名\t年龄\t学号\n");
    foreach ($students as $key => $val) {
        printf("%d\t%s\t%d\t%d\n", $key, $val['name'], $val['age'], $val['number']);
    }
?>
```

效果演示：

```bash
[vagrant@localhost default]$ ./stdio_test.php
请输入学生的姓名、年龄和学号：张三 23 345353
请输入学生的姓名、年龄和学号：李四 22 342423
请输入学生的姓名、年龄和学号：王五 22 234243
请输入学生的姓名、年龄和学号：赵六 21 234254

# 输出内容
序号    姓名    年龄    学号
1       张三    23      345353
2       李四    22      342423
3       王五    22      234243
4       赵六    21      234254
```

PHP 实现文件复制：

```php
#!/usr/bin/env php
<?php
    if (!defined('READ_SIZE')) {
        define('READ_SIZE', 4096);
    }

    function _error_handle(string $msg)
    {
        printf("\033[0;31;43m%s\033[0m\n", $msg);
        exit();
    }

    if ($argc < 3) {
        _error_handle("Missing file operand");
    }

    if ($argv[1] === $argv[2]) {
        _error_handle("'{$argv[1]}' and '{$argv[2]}' are the same file.");
    }

    if (($fp_read = @fopen($argv[1], 'rb')) === false || ($fp_write = @fopen($argv[2], 'wb')) == false) {
        _error_handle("Fail to open file.");
    }

    while (!feof($fp_read)) {
        $content = fread($fp_read, READ_SIZE);
        fwrite($fp_write, $content, strlen($content));
    }
    fclose($fp_read);
    fclose($fp_write);
?>
```

效果演示：

```bash
# 文件复制
[vagrant@localhost default]$ ./copy.php 蓝莲花.ncm lanlianhua.ncm

# 查看复制后的文件
[vagrant@localhost default]$ ls -al ./蓝莲花.ncm ./lanlianhua.ncm
-rwxrwxrwx 1 vagrant vagrant 11116126 Oct 15 14:19 ./lanlianhua.ncm
-rwxrwxrwx 1 vagrant vagrant 11116126 Jun 16 15:59 ./蓝莲花.ncm
```
