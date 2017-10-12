---
title: tcc源码解析系列(三)之启动详解
date: 2017-10-12 16:03:53
categories: happylifeplat-tcc
---


### 启动源码详解

*  通过上面的二篇文章，我相信您对tcc应该有个大体的了解，并且已经搭建好了调试环境，那么就让我们一起探索tcc的源码之旅。

* 首先还记得我们在项目中applicationContext.xml有一段这么的配置吗?
```xml
<!-- Aspect 切面配置，是否开启AOP切面-->
  <aop:aspectj-autoproxy expose-proxy="true"/>
  <!--扫描框架的包-->
  <context:component-scan base-package="com.happylifeplat.tcc.*"/>
  <!--启动类属性配置-->
  <bean id="tccTransactionBootstrap" class="com.happylifeplat.tcc.core.bootstrap.TccTransactionBootstrap">
         <property name="serializer" value="kryo"/>
         <property name="coordinatorQueueMax" value="5000"/>
         <property name="coordinatorThreadMax" value="4"/>
         <property name="recoverDelayTime" value="120"/>
         <property name="retryMax" value="3"/>
         <property name="rejectPolicy" value="Abort"/>
         <property name="blockingQueueType" value="Linked"/>
         <property name="scheduledDelay" value="120"/>
         <property name="scheduledThreadMax" value="4"/>
         <property name="repositorySupport" value="db"/>
         <property name="tccDbConfig">
             <bean class="com.happylifeplat.tcc.common.config.TccDbConfig">
                 <property name="url"
                           value="jdbc:mysql://192.168.1.68:3306/account?useUnicode=true&amp;characterEncoding=utf8"/>
                 <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
                 <property name="password" value="Wgj@555888"/>
                 <property name="username" value="xiaoyu"/>
             </bean>
         </property>
     </bean>
```
* 通过以上的配置我们知道首先需要开启Aop切面，再扫描框架的包，重点我们来关注 TccTransactionBootstrap
