---
layout: post
title: "微服务系列一：基于grpc-gateway创建网关swagger文档服务"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-part7.jpg
categories: "grpc"
date: 2023-12-07 11:45:56
tags: ['grpc', 'go']
---

## 概述

`Swagger`是全球最大的OpenAPI规范（OAS）API开发工具框架，支持从设计和文档到测试和部署的整个API生命周期的开发。Swagger是目前最受欢迎的RESTful Api文档生成工具之一，主要的原因如下：

- 跨平台、跨语言的支持
- 强大的社区
- 生态圈 Swagger Tools（Swagger Editor、Swagger Codegen、Swagger UI ...）
- 强大的控制台
- 同时`grpc-gateway`也支持`Swagger`。

本文展示了`gRPC-Gateway`集成`swagger`的常规流程，由以下步骤组成：

1. 新建工程文件夹；
2. 安装必要的go包；
3. 编写proto文件，使swagger支持http（默认是https）；
4. 生成gRPC、gRPC-Gateway所需的go源码；
5. 生成swagger所需的json文件；
6. 下载swagger-ui的源码，以此生成go源码；
7. 编写gRPC的服务端代码；
8. 编写gRPC-Gateway服务端的代码;
9. 验证

# 环境配置

## 安装 grpc，grpc-Gateway 插件

下载最新的grpc-gateway插件：

```bash
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
```

## 安装配置 protoc-gen-openapiv2 插件

下载当前最新稳定版本的protoc-gen-openapi v2插件，用于生成swagger-ui要用的json文件，依据此文件，swagger才能正确的展现出gRPC-Gateway暴露的服务和参数定义：

```bash
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```

## 安装 go-bindata

下载当前最新稳定版本go-bindata, 用于将swagger-ui的源码转为GO代码：

```bash
go install -a -v github.com/go-bindata/go-bindata/v3/...@latest
```

swagger-ui的代码由多个png、html、js文件组成，需要用工具go-bindata转换成go源码并放入合适的位置，流程如下图：　　

## 安装go-bindata-assetfs

下载当前最新稳定版本`go-bindata-assetfs`,在应用启动后，对外提供文件服务，这样可以通过web访问swagger的json文件：

```bash
go install github.com/elazarl/go-bindata-assetfs/...@latest
```

# Swagger网关实战

