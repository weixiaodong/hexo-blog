---
title: go-每日一库之-cobra-cli使用
date: 2023-03-10 20:59:22
tags:
---

## cobra命令生成工具

安装

```bash
go install github.com/spf13/cobra-cli@latest
```

<!--more-->

## go项目初始化

```bash
cd $HOME/code 
mkdir myapp
cd myapp
go mod init github.com/spf13/myapp
```

## 初始化 Cobra CLI 应用程序

```bash
cd $HOME/code/myapp
cobra-cli init
go run main.go

```

## 向项目添加命令

```bash
cobra-cli add serve
cobra-cli add config
cobra-cli add create -p 'configCmd'
```

## 配置 Cobra 生成器

示例 ~/.cobra.yaml 文件：

```yaml
author: Steve Francia <spf@spf13.com>
license: MIT
useViper: true

```