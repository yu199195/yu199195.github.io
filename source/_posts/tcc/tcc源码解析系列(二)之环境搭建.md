---
title: tcc源码解析系列(二)之环境搭建
date: 2017-10-11 16:03:53
categories: hmily-tcc
permalink: TCC/tcc-two
---

### 环境搭建教程
在上一篇中，我们了解了项目的整体结构，以及每个模块大概的作用，现在我们来开始搭建整个环境。

* 首先用户使用的JDK必须是1.8+  本地安装了git ,maven ，执行以下命令

```
git clone  https://github.com/yu199195/hmily.git

maven clean install
```

* 使用你的开发工具打开项目，比如idea Eclipse

* 打开你的数据库工具，执行工程文件sql文件夹下的[tcc-demo.sql](https://github.com/yu199195/hmily/blob/master/hmily-tcc-demo/sql/tcc-demo.sql)


### dubbo 用户

* 进入hmily-tcc-demo-dubbo-account项目，修改application.yml中的数据库配置，如下图：

  ![](https://yu199195.github.io/images/hmily-tcc/02.png)

* 修改applicationContext.xml中的配置，具体可以参考 [配置详解](https://github.com/yu199195/hmily/wiki/Configuration)

  ![](https://yu199195.github.io/images/hmily-tcc/03.png)

* 修改spring-dubbo.xml 中的zookeeper配置,如图所示:

  ![](https://yu199195.github.io/images/hmily-tcc/04.png)

*  inventory,order项目的配置修改和上面的一样，注意dubbo的端口不要重复。

* 依次执行AccountApplication，InventoryApplication,OrderApplication中的main方法

* 访问http://localhost:8083/swagger-ui.html 进入体验体验dubbo的分布式事务。


### springcloud用户

* 修改各项目中的application.yml的数据库配置。

 ![](https://yu199195.github.io/images/hmily-tcc/05.png)

* 修改各项目中applicationContext.xml的配置，具体可以参考 [配置详解](https://github.com/yu199195/hmily/wiki/Configuration)

    ![](https://yu199195.github.io/images/hmily-tcc/03.png)

* 执行hmily-tcc-demo-springcloud-eureka项目中的EurekaServerApplication类的main方法

* 依次执行AccountApplication，InventoryApplication,OrderApplication中的main方法

* 访问http://localhost:8884/swagger-ui.html 进入体验Springcloud分布式事务。


### 如有任何问题欢迎加入QQ群：162614487 进行讨论
