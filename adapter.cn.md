---
layout: default
title: 标准化 RPC
---

标准化 RPC

* TOC
{:toc}

一个WEB框架基本上都需要包含以下四个方面的功能。现有的 WEB 框架的模式存在以下的问题：

* 因为和具体实现紧耦合。像日志库这样的东西很多地方都要用，但是不想引入那么重的依赖。提供 interface 的模式可以方便用户组合模块的时候，不会碰到用了四五种不同日志库的尴尬。
* 和具体的协议耦合，HTTP和THRIFT等不同接入方式无法复用代码。
* 各种 RPC 没有统一的切入点，不同的 RPC 调用要重复写代码接入服务发现，熔断，metrics上报等代码。
* Go 裸写 SQL 的体验非常差。beego 等框架自带的 ORM 又不满足很多场景的需求。SQL 需要一个简单易用，类似 ibatis 的裸写方式。

# Logging

提供标准化的 API/SPI 接口，适配各种现有的日志库。

# Server

HTTP/THRIFT/MQ 三种服务接入方式，适配成统一的接口。

# Client

HTTP/THRIFT等RPC服务，统一适配成 client 的抽象。

# SQL

把 Go 的 SQL 标准库适配成一个更易于使用的接口。
