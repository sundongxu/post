---
title: "Go Modules 简单介绍"
date: 2019-01-14T00:24:08+08:00
categories:
    - Golang
tags: 
    - Golang
    - module
---

# Go Modules

Golang 在 1.11 版本推出了万众期待的依赖管理工具 [go Modules](https://github.com/golang/go/wiki/Modules)

我厌倦了 Glide 那无穷无尽的 update 状态之后，还是决定自己再一次尝试新的依赖管理工具 (~~ 毕竟是 google 爸爸推出的~~)

还好尝试的比较晚，已经有许多 bug 得到解决，社区也有许多文章给出了最佳实践，建议先看看 [intro-to-go-modules](https://roberto.selbach.ca/intro-to-go-modules/)

## 1. enable GO111MODULE
go modules 默认是没有开启的，需要设置环境变量 `GO111MODULE=on`, 如果没有设置, 会有一些提示。

## 2. help doc
接下来就是查看一下帮助手册 `go mod help`:

```golang
Go mod provides access to operations on modules.

Note that support for modules is built into all the go commands,
not just 'go mod'. For example, day-to-day adding, removing, upgrading,
and downgrading of dependencies should be done using 'go get'.
See 'go help modules' for an overview of module functionality.

Usage:

	go mod <command> [arguments]

The commands are:

	download    download modules to local cache
	edit        edit go.mod from tools or scripts
	graph       print module requirement graph
	init        initialize new module in current directory
	tidy        add missing and remove unused modules
	vendor      make vendored copy of dependencies
	verify      verify dependencies have expected content
	why         explain why packages or modules are needed

Use "go help mod <command>" for more information about a command.
```

简单介绍一下命令的功能:
- download 下载依赖的 module 到本地 cache
- edit 编辑 go.mod 文件
- graph 打印现有 module 依赖图
- init 当前文件夹下初始化一个新的 module, 创建 go.mod 文件
- tidy 增加丢失的 module，去掉没有使用的 module
- vendor 将 module 依赖复制到 vendor 下
- verify 校验依赖
- why 为什么需要依赖

`go.mod` 文件一旦创建后，它的内容将会被 `go toolchain` 管理。`go toolchain` 会在比如 `go get`、`go build`、`go mod` 等命令修改和维护 `go.mod` 文件

## 3. init go modules
对于已经存在的项目应该这样初始化:
- `cd {your project go path}`
- `go mod init` 创建一个空的 `go.mod`
- `go get ./...` 查找依赖，并记录在 `go.mod` 文件中
- `go mod tidy` 增加丢失的依赖，删除不需要的依赖

对于已经新项目应该这样初始化:
- `go mod init {your project path}`  创建一个空的 `go.mod`
- `go mod tidy` 或者 手动维护 `go.mod` 里面的 require

## 4. tips
你可以在 `GOPATH` 之外创建新的项目

`go mod download` 可以下载所需要的依赖，但是依赖并不是下载到 `$GOPATH` 中，而是 `$GOPATH/pkg/mod` 中，多个项目可以共享缓存的 module

可以手动编辑 `go.mod` 的 replace 将需要翻墙的包替换成 github 镜像

```
replace (
    golang.org/x/text v0.3.0 => github.com/golang/text v0.3.0
)
```

## 5. go get
可以运行 `go get` 升级需要的依赖

- `go get -u` 将会升级到最新的次要版本或者修订版本
- `go get -u=patch` 将会升级到最新的修订版本
- `go get package@version` 将会升级到指定的版本号 `version`

## 6. version tag
`go mod` 目前支持的 version tag

- "gopkg.in/check.v1" v0.0.0-20161208181325-20d25e280405
- "gopkg.in/yaml.v2" <=v2.2.2
- "gopkg.in/yaml.v2" latest
