---
title: "C语言中 Sizeof 和 Strlen 的区别"
date: 2021-10-12T11:10:54+08:00
draft: true
tags: ["C语言"]
categories: ["C语言"]
---

最近遇到一个比较奇怪的事情，我的字符数组明明只有10个字符，可是我用 strlen 函数计算字符数组长度时，却显示长度为22。实在想不明白，后来 Google 了一番，原来 C 语言中字符串默认是以 '\0' 结尾，strlen 在计算字符串长度时，遇到 '\0' 就结束。我的字符数组在初始化时长度刚好能放下字符串，却忽略了结尾的 '\0'，最终导致 strlen 计算长度时，会出现一些莫名其妙的结果。哎，还是自己基础知识掌握不牢靠呀！写篇文章，记录一下吧。

<!--more-->

## 代码示例

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char const *argv[])
{
    char *str1 = "hello world";
    char str2[] = "hello world";
    char str3[] = {"hello world"};
    char str4[] = {'h', 'e', 'l', 'l', 'o', ' ', 'w', 'o', 'r', 'l', 'd'};
    char str5[] = {'h', 'e', 'l', 'l', 'o', ' ', 'w', 'o', 'r', 'l', 'd', '\0'};
    char str6[11] = "hello world";
    char str7[12] = "hello world";
    char str8[10] = "hello";

    // 注意：每个人的运行结果可能会不一样，此处只做演示。可以看到 str4 和 str6 的长度为 22，strlen 函数会直到遇见 '\0' 才会停止。
    printf("strlen(str1) = %zd, sizeof(str1) = %zd\n", strlen(str1), sizeof(str1)); // strlen(str1) = 11, sizeof(str1) = 8
    printf("strlen(str2) = %zd, sizeof(str2) = %zd\n", strlen(str2), sizeof(str2)); // strlen(str2) = 11, sizeof(str2) = 12
    printf("strlen(str3) = %zd, sizeof(str3) = %zd\n", strlen(str3), sizeof(str3)); // strlen(str3) = 11, sizeof(str3) = 12
    printf("strlen(str4) = %zd, sizeof(str4) = %zd\n", strlen(str4), sizeof(str4)); // strlen(str4) = 22, sizeof(str4) = 11
    printf("strlen(str5) = %zd, sizeof(str5) = %zd\n", strlen(str5), sizeof(str5)); // strlen(str5) = 11, sizeof(str5) = 12
    printf("strlen(str6) = %zd, sizeof(str6) = %zd\n", strlen(str6), sizeof(str6)); // strlen(str6) = 22, sizeof(str6) = 11
    printf("strlen(str7) = %zd, sizeof(str7) = %zd\n", strlen(str7), sizeof(str7)); // strlen(str7) = 11, sizeof(str7) = 12
    printf("strlen(str8) = %zd, sizeof(str8) = %zd\n", strlen(str8), sizeof(str8)); // strlen(str8) = 5, sizeof(str8) = 10

    return 0;
}
```

## sizeof 

* sizeof 是 C 语言中的一元运算符，用于计算其操作数所占内存的大小，编译器在编译时就计算出了 sizeof 的结果。
* sizeof 可以用于任何数据类型，包括原始类型（例如整型和浮点类型）、指针类型（例如上面示例中的 str1）或复合类型（例如struct、union等）

## strlen

* strlen 是 C 语言中的预定义函数，其定义包含在头文件 string.h 中。
* strlen 接受一个指向字符数组的指针作为参数，并在运行时从我们给它的地址遍历内存，寻找一个 '\0' 字符，并计算它在找到之前一共通过了多少内存位置。
* strlen 的主要任务是计算数组或字符串的长度。（例如上面示例中的 str8）

## strlen 和 sizeof 的区别

1. sizeof 是运算符，strlen 是预定义函数。
2. sizeof 以字节（包括 \0）为单位给出任意类型数据（已分配）的实际大小。
3. sizeof 是一个编译时表达式，它用以计算类型或变量类型的大小，它不关心变量的值；strlen 用以计算以 '\0' 结尾的字符串的长度。

