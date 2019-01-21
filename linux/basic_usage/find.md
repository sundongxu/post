---
title: "Linux 常用命令 find"
date: 2019-01-21T22:39:31+08:00
categories:
    - Linux
tags: 
    - Linux
    - tips
---

查找指定文件名的文件(不区分大小写)
```
find -iname [filename]
```

对找到的文件执行某个命令
```
find -iname [filename] -exec md5sum {} \;
```

查找目录下的所有空文件
```
find [dir] -empty
```

按照文件权限来查找文件
```
find [dir] -perm [mode] –print
```

按照文件的更改时间来查找文件， `-n`表示文件更改时间距现在n天以内，`+n`表示文件更改时间距现在n天以前
```
find [dir] -mtime -5 –print
```

查找更改时间比文件file1新但比文件file2旧的文件
```
find [dir] -newer file1 ! file2 
```

根据类型查找 `-type`
```
find [dir] -type d –print
```
> b - 块设备文件
d - 目录
c - 字符设备文件
p - 管道文件
l - 符号链接文件
f - 普通文件

忽略某个目录
```
find [dir] -path [exclude_dir] -prune -o -print
```

在目录下查找文件长度大于1 M字节的文件
```
find [dir] -size +1000000c -print
```

在查找文件时，首先查找当前目录中的文件，然后再在其子目录中查找
```
find [dir] -name [filename] -depth –print
```