---
title: tcc源码解析系列(一)之项目结构
date: 2017-10-11 15:03:53
categories: happylifeplat-tcc
permalink: TCC/tcc-one
---

### [happylifeplat-tcc](https://github.com/yu199195/happylifeplat-tcc) 是什么？有什么功能？
  *  这是碧桂园旺生活解决分布式事务的TCC开源方案。[github地址](https://github.com/yu199195/happylifeplat-tcc)
  * 支持dubbo，springcloud等rpc框架进行分布式事务
  *  本地事务存储，支持redis，mogondb，zookeeper，file，mysql等关系型数据库
  * 序列化方式，支持java，hessian，kryo，protostuff

###  项目结构
![结构图](https://yu199195.github.io/images/happylifeplat-tcc/01.png)

*  happylifeplat-annotation 提供分布式事务的@Tcc注解,对于向dubbo这种面向接口的rpc框架，为了保证接口的轻量性，所以抽离出来，单独做为一个项目。

* happylifeplat-tcc-common 从名称可以看出是tcc框架的一个公共项目，里面主要是一些配置，枚举，异常定义等。

* happylifeplat-tcc-core 该项目是tcc框架的核心实现，包括服务的启动，调用流程，aop切面，重试等实现。

* happylifeplat-tcc-dubbo  该项目是对dubbo框架的支持，里面主要针对dubbo的特性的实现。

* happylifeplat-tcc-springcloud 该项目是对springcloud框架的支持，里面主要针对springcloud的特性的实现。

* happylifeplat-tcc-demo 这是实战体验的demo项目，里面有针对dubbo用户和springcloud用户的案列，里面具体的配置，用户可以参考 [dubbo用户](https://github.com/yu199195/happylifeplat-tcc/wiki/%E5%BF%AB%E9%80%9F%E4%BD%93%E9%AA%8C%EF%BC%88dubbo%EF%BC%89)  ,    [springcloud用户](https://github.com/yu199195/happylifeplat-tcc/wiki/%E5%BF%AB%E9%80%9F%E4%BD%93%E9%AA%8C%EF%BC%88springcloud%EF%BC%89)
