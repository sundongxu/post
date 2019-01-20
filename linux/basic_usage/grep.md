---
title: "Linux 常用命令 grep"
date: 2019-01-20T22:28:19+08:00
categories:
    - Linux
tags: 
    - Linux
    - tips
---


在文件中查找字符串(不区分大小写)
```
grep -i [word] [file]
```

在一个文件夹中递归查询包含指定字符串的文件
```
grep -r [word] [file / path]
```

从文件中读取关键词进行搜索
```
cat [file] | grep -f [word]
```

从文件中读取关键词进行搜索 且显示行号
```
cat [file] | grep -nf [word]
```

从多个文件中查找关键词
```
grep [word] [file1] [file2]
```

显示包含 a 或者 b 字符的内容行
```
cat [file] |grep -E "a|b"
```

grep 不显示本身进程

```
ps aux | grep [word] | grep -v "grep"
```

使用 `^` 符号输出所有以某指定模式开头的行

```
grep ^docker [file]
```

使用 `$` 符号输出所有以指定模式结尾的行
```
grep git$ [file]
```

使用 grep 查找文件中所有的空行
```
grep ^$ [file]
```

输出匹配指定模式行的前N行
```
grep -A [N] [word] [file]
```

输出匹配指定模式行的后N行
```
grep -B [N] [word] [file]
```

输出匹配指定模式行的前后各N行
```
grep -C [N] [word] [file]
```

查找指定进程
```
 ps -ef | grep [word]
```
> 最后一条结果是grep进程本身，并非真正要找的进程

查找指定进程个数
```
ps -ef | grep -c [word]
```