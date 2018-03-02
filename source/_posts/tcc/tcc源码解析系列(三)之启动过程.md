---
title: tcc源码解析系列(三)之启动详解
date: 2017-10-12 17:03:53
categories: hmily-tcc
permalink: TCC/tcc-three
---


### 启动源码详解

*  通过上面的二篇文章，我相信您对tcc应该有个大体的了解，并且已经搭建好了调试环境，那么就让我们一起探索tcc的源码之旅。


* 首先看任何框架的源码都需要找到框架的入口，tcc也不例外，还记得我们在项目中applicationContext.xml有一段这么的配置吗?
```xml
<!-- Aspect 切面配置，是否开启AOP切面-->
  <aop:aspectj-autoproxy expose-proxy="true"/>
  <!--扫描框架的包-->
  <context:component-scan base-package="com.hmily.tcc.*"/>
  <!--启动类属性配置-->
  <bean id="tccTransactionBootstrap" class="com.hmily.tcc.core.bootstrap.TccTransactionBootstrap">
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
             <bean class="com.hmily.tcc.common.config.TccDbConfig">
                 <property name="url"
                           value="jdbc:mysql://192.168.1.68:3306/account?useUnicode=true&amp;characterEncoding=utf8"/>
                 <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
                 <property name="password" value="Wgj@555888"/>
                 <property name="username" value="xiaoyu"/>
             </bean>
         </property>
     </bean>
```
* 通过以上的配置我们知道首先需要开启Aop切面，再扫描框架的包，重点我们来关注 **TccTransactionBootstrap**

### TccTransactionBootstrap 源码解析
```java
package com.hmily.tcc.core.bootstrap;

import com.hmily.tcc.common.config.TccConfig;
import com.hmily.tcc.core.helper.SpringBeanUtils;
import com.hmily.tcc.core.service.TccInitService;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.stereotype.Component;


@Component
public class TccTransactionBootstrap extends TccConfig implements ApplicationContextAware {


    private final TccInitService tccInitService;

    @Autowired
    public TccTransactionBootstrap(TccInitService tccInitService) {
        this.tccInitService = tccInitService;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
       //保存spring的上下文
        SpringBeanUtils.getInstance().setCfgContext((ConfigurableApplicationContext) applicationContext);
        start(this);
    }


    private void start(TccConfig tccConfig) {
        tccInitService.initialization(tccConfig);
    }
}
```
* 它继承 **TccConfig** 能获取在xml配置的属性信息，实现 **ApplicationContextAware** 当spring容器初始化的时候，会自动的将ApplicationContext注入进来

* 我们继续跟踪代码，进入**initialization** 方法
```java
public void initialization(TccConfig tccConfig) {
       Runtime.getRuntime().addShutdownHook(new Thread(() -> LOGGER.error("系统关闭")));
       try {
           //加载spi配置，把spi的配置注入成spring的bean 方便后续的使用
           //就是框架所支持的序列化，存储方式
           LoadSpiSupport(tccConfig);
           coordinatorService.start(tccConfig);
       } catch (Exception ex) {
           LogUtil.error(LOGGER, "tcc事务初始化异常:{}", ex::getMessage);
           System.exit(1);//非正常关闭
       }
       LogUtil.info(LOGGER, () -> "Tcc事务初始化成功！");
   }
```
* **LoadSpiSupport** 采用jdk自带的spi加载，如果有不明白的小伙伴，可以自行google

* 我们继续进入 **coordinatorService.start(tccConfig)**
``` java
@Override
  public void start(TccConfig tccConfig) throws Exception {
      this.tccConfig = tccConfig;
      //获取应用名称
      final String appName = applicationService.acquireName();
      //获取上一步加载的spi资源信息
      coordinatorRepository = SpringBeanUtils.getInstance().getBean(CoordinatorRepository.class);
      //初始化spi 协调资源存储
      coordinatorRepository.init(appName, tccConfig);
      //初始化 协调资源线程池
      initCoordinatorPool();
      //定时执行补偿
      scheduledRollBack();
  }
```
*  **coordinatorRepository.init(appName, tccConfig)** 就是根据spi思想来具体初始化，现在支持的如图：
  ![](https://yu199195.github.io/images/hmily-tcc/06.png)

*   **initCoordinatorPool()** 初始化 协调资源线程池
```java
    private void initCoordinatorPool() {
        synchronized (LOGGER) {
            //采用LinkedBlockingQueue
            QUEUE = new LinkedBlockingQueue<>(tccConfig.getCoordinatorQueueMax());
            final int coordinatorThreadMax = tccConfig.getCoordinatorThreadMax();
            final TccTransactionThreadPool threadPool = SpringBeanUtils.getInstance().getBean(TccTransactionThreadPool.class);
            //获取固定数量线程大小的线程池
            final ExecutorService executorService = threadPool.newCustomFixedThreadPool(coordinatorThreadMax);
            LogUtil.info(LOGGER, "启动协调资源操作线程数量为:{}", () -> coordinatorThreadMax);
            for (int i = 0; i < coordinatorThreadMax; i++) {
               //执行线程
                executorService.execute(new Worker());
            }

        }
    }
    class Worker implements Runnable {

       @Override
       public void run() {
           execute();
       }

       private void execute() {
           while (true) {
               try {
                  //阻塞队列获取
                   final CoordinatorAction coordinatorAction = QUEUE.take();
                   if (coordinatorAction != null) {
                       final int code = coordinatorAction.getAction().getCode();
                       if (CoordinatorActionEnum.SAVE.getCode() == code) {
                           save(coordinatorAction.getTccTransaction());
                       } else if (CoordinatorActionEnum.DELETE.getCode() == code) {
                           remove(coordinatorAction.getTccTransaction().getTransId());
                       } else if (CoordinatorActionEnum.UPDATE.getCode() == code) {
                           update(coordinatorAction.getTccTransaction());
                       }
                   }
               } catch (Exception e) {
                   e.printStackTrace();
                   LogUtil.error(LOGGER, "执行协调命令失败：{}", e::getMessage);
               }
           }

       }
   }
```

*  **scheduledRollBack()** 执行定时补偿，这个以后再详细讲解逻辑


### 到此tcc框架的初始化就已经完成的启动，是不是很简单？如有任何问题或者建议欢迎加入QQ群：162614487 进行讨论
