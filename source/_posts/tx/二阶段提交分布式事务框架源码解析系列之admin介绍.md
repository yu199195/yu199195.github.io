---
title: 二阶段提交分布式事务框架源码解析系列之transaction-admin 事务跟踪查看管理后台
date: 2017-10-14 11:03:53
categories: Raincat
permalink: tx/tx-seven
---

### transaction-admin 启动教程
启动前提：分布式事务项目已经部署并且运行起来，用户根据自己的RPC框架进行使用
[dubbo 用户](https://github.com/yu199195/Raincat/wiki/dubbo%E7%94%A8%E6%88%B7%E6%8C%87%E5%8D%97)
[springcloud 用户](https://github.com/yu199195/Raincat/wiki/springcloud%E7%94%A8%E6%88%B7%E6%8C%87%E5%8D%97)


* 首先用户使用的JDK必须是1.8+  本地安装了git ,maven ，执行以下命令

```
git clone https://github.com/yu199195/Raincat.git

maven clean install
```

* 使用你的开发工具打开项目，比如idea Eclipse

### 步揍一：  配置并且启动transaction-admin
* 在项目中的application.properties文件中，修改您的服务端口，redis配置等配置:

```java
#admin项目的tomcat端口
server.port=8888
#admin项目的上下文
server.context-path=/admin
server.address=0.0.0.0
#admin项目的应用名称
spring.application.name=tx-transaction-admin

# admin项目激活的类型，支持db，file，mongo，zookeeper，redis 下文会继续讲解
spring.profiles.active=db

# txManager redis 配置
#集群配置
#tx.redis.cluster=true
#tx.redis.cluster.nodes=127.0.0.1:70001;127.0.1:7002
#tx.redis.cluster.redirects=20

#单机配置
tx.redis.cluster=false
tx.redis.hostName=192.168.1.68
#redis主机端口
tx.redis.port=6379
#tx.redis.password=happylifeplat01


# admin管理后台的用户名，用户可以自己更改
tx.admin.userName=admin
# admin管理后台的密码，用户可以自己更改
tx.admin.password=admin


#采用二阶段提交项目的应用名称集合，这个必须要填写
recover.application.list=alipay-service,wechat-service,pay-service

#事务补偿最大重试次数
recover.retry.max=10

#事务补偿的序列方式
recover.serializer.support=kryo

#dbSuport 采用db进行的补偿方式的配置
recover.db.driver=com.mysql.jdbc.Driver
recover.db.url=jdbc:mysql://192.168.1.68:3306/tx?useUnicode=true&amp;characterEncoding=utf8
recover.db.username=xiaoyu
recover.db.password=Wgj@555888

#redis 采用redis进行补偿方式的配置
recover.redis.cluster=false
recover.redis.hostName=192.168.1.68
recover.redis.port=6379
recover.redis.password=
#recover.redis.clusterUrl=127.0.0.1:70001;127.0.1:7002

#mongo 采用mongo 进行补偿方式的配置
recover.mongo.url=192.168.1.68:27017
recover.mongo.dbName=happylife
recover.mongo.userName=xiaoyu
recover.mongo.password=123456

#zookeeper 采用zookeeper进行补偿的方式配置
recover.zookeeper.host=192.168.1.132:2181
recover.zookeeper.sessionTimeOut=200000


```

## 配置解释

* 关于txManager的redis配置:其实就是在txManager项目中的redis配置，在这里需要配置成一样的；
注意，如果你的redis是集群模式请参考集群配置，并注释掉单机redis配置，同理redis单机模式也一样。


* 关于 recover.application.list配置：这里需要配置每个参与二阶段分布式事务的系统模块的applicationName，多个模块用 "," 分隔，这里必须要配置。

* 关于 recover.serializer.support 配置，这里是指参与二阶段分布式事务系统中，配置事务补偿信息的序列化方式。

* 关于 spring.profiles.active 配置 admin项目激活的类型，支持db，file，mongo，zookeeper，
  这里是指参与二阶段分布式事务系统中，配置事务补偿信息存储方式，如果您用db存储，那这里就配置成db，同时配置好 recover.db等信息。 其他方式同理。 注意，每个模块请使用相同的序列化方式和存储类型

* 关于 tx.admin 等配置。 这里就是管理后台登录的用户与密码，用户可以进行自定义更改。


### 步揍二：修改 index.html

```html
<!--href 修改成你的ip 端口-->
<a id="serverIpAddress" style="display: none" href="http://192.168.1.132:8888/admin">
```


### 步揍三: 运行 AdminApplication 中的main方法。


### 步揍四:在浏览器访问  http://ip:prot/admin  ,输入用户名，密码登录。


![登录界面](https://yu199195.github.io/images/Raincat/txlogin.png)


![首页](https://yu199195.github.io/images/Raincat/txIndex.png)

![事务组](https://yu199195.github.io/images/Raincat/txGroupInfo.png)

![事务补偿](https://yu199195.github.io/images/Raincat/txRecoverInfo.png)





### 如有任何问题欢迎加入QQ群：162614487 进行讨论
