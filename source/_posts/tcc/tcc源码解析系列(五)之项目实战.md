---
title: tcc源码解析系列(五)之项目实战
date: 2017-10-12 19:03:53
categories: hmily-tcc
permalink: TCC/tcc-five
---

#### 接上一篇，我们已经分析了在整个消费的调用流程，现在只差发起真实的rpc远端调用了，这篇文章，我们一起进入提供者的调用流程吧！


* 我们发起 **accountService.payment(accountDTO);** 的调用，在提供方，我们可以看到其实现类为AccountServiceImpl:

```java

/**
    * 扣款支付
    *
    * @param accountDTO 参数dto
    * @return true
    */
   @Override
   @Tcc(confirmMethod = "confirm", cancelMethod = "cancel")
   public boolean payment(AccountDTO accountDTO) {
       final AccountDO accountDO = accountMapper.findByUserId(accountDTO.getUserId());
       accountDO.setBalance(accountDO.getBalance().subtract(accountDTO.getAmount()));
       accountDO.setFreezeAmount(accountDO.getFreezeAmount().add(accountDTO.getAmount()));
       accountDO.setUpdateTime(new Date());
       final int update = accountMapper.update(accountDO);
       if (update != 1) {
           throw new TccRuntimeException("资金不足！");
       }
       return Boolean.TRUE;
   }

   public boolean confirm(AccountDTO accountDTO) {

       LOGGER.debug("============执行确认付款接口===============");

       final AccountDO accountDO = accountMapper.findByUserId(accountDTO.getUserId());
       accountDO.setFreezeAmount(accountDO.getFreezeAmount().subtract(accountDTO.getAmount()));
       accountDO.setUpdateTime(new Date());
       accountMapper.update(accountDO);
       return Boolean.TRUE;
   }


   public boolean cancel(AccountDTO accountDTO) {

       LOGGER.debug("============执行取消付款接口===============");
       final AccountDO accountDO = accountMapper.findByUserId(accountDTO.getUserId());
       accountDO.setBalance(accountDO.getBalance().add(accountDTO.getAmount()));
       accountDO.setFreezeAmount(accountDO.getFreezeAmount().subtract(accountDTO.getAmount()));
       accountDO.setUpdateTime(new Date());
       accountMapper.update(accountDO);
       return Boolean.TRUE;
   }

```
* 我们发现它也有@Tcc注解，并且提供了confrim，cancel等真实的方法。通过前面一篇的分析，我们知道，他是springBean的一个实现类，同样会走切面。

* 经过 **TccTransactionFactoryServiceImpl** 的 factoryOf方法，我们可以知道他会返回 **ProviderTccTransactionHandler**

```java
@Override
   public Class factoryOf(TccTransactionContext context) throws Throwable {

       //如果事务还没开启或者 tcc事务上下文是空， 那么应该进入发起调用
       if (!tccTransactionManager.isBegin() && Objects.isNull(context)) {
           return StartTccTransactionHandler.class;
       } else if (tccTransactionManager.isBegin() && Objects.isNull(context)) {
           return ConsumeTccTransactionIHandler.class;
       } else if (Objects.nonNull(context)) {
           return ProviderTccTransactionHandler.class;
       }
       return ConsumeTccTransactionIHandler.class;
   }
```

* 最终我们来到 **ProviderTccTransactionHandler.handler** 方法：

```java
/**
    * 分布式事务提供者处理接口
    * 根据tcc事务上下文的状态来执行相对应的方法
    *
    * @param point   point 切点
    * @param context context
    * @return Object
    * @throws Throwable 异常
    */
   @Override
   public Object handler(ProceedingJoinPoint point, TccTransactionContext context) throws Throwable {
       TccTransaction tccTransaction = null;
       try {
           switch (TccActionEnum.acquireByCode(context.getAction())) {
               case TRYING:
                   try {
                       //创建事务信息
                       tccTransaction = tccTransactionManager.providerBegin(context);
                       //发起方法调用
                       return point.proceed();
                   } catch (Throwable throwable) {
                       tccTransactionManager.removeTccTransaction(tccTransaction);
                       throw throwable;

                   }
               case CONFIRMING:
                   //如果是confirm 通过之前保存的事务信息 进行反射调用
                   final TccTransaction acquire = tccTransactionManager.acquire(context);
                   tccTransactionManager.confirm();
                   break;
               case CANCELING:
                   //如果是调用CANCELING 通过之前保存的事务信息 进行反射调用
                   tccTransactionManager.acquire(context);
                   tccTransactionManager.cancel();
                   break;
               default:
                   break;
           }
       } finally {
           tccTransactionManager.remove();
       }
       Method method = ((MethodSignature) (point.getSignature())).getMethod();
       return getDefaultValue(method.getReturnType());
   }

```

* **TccTransactionContext** 就是通过rpc json序列化后传过来的对象，此时我们知道是在try阶段，所以我们进入try

```java
try {
    //创建事务信息
    tccTransaction = tccTransactionManager.providerBegin(context);
    //发起方法调用
    return point.proceed();
} catch (Throwable throwable) {
    tccTransactionManager.removeTccTransaction(tccTransaction);
    throw throwable;

}
```
* 首先我们会创建提供者的事务信息，并把他存起来，再把它存入threadlocal中，接着发起  **point.proceed()** 调用的时候，我们会进入
**TccCoordinatorMethodAspect**,由于是在try阶段最终会进入：

```java
/**
    * 获取调用接口的协调方法并封装
    *
    * @param point 切点
    */
   private void registerParticipant(ProceedingJoinPoint point, String transId) throws NoSuchMethodException {

       MethodSignature signature = (MethodSignature) point.getSignature();
       Method method = signature.getMethod();

       Class<?> clazz = point.getTarget().getClass();

       Object[] args = point.getArgs();

       final Tcc tcc = method.getAnnotation(Tcc.class);

       //获取协调方法
       String confirmMethodName = tcc.confirmMethod();

      /* if (StringUtils.isBlank(confirmMethodName)) {
           confirmMethodName = method.getName();
       }*/

       String cancelMethodName = tcc.cancelMethod();

      /* if (StringUtils.isBlank(cancelMethodName)) {
           cancelMethodName = method.getName();
       }
*/
       //设置模式
       final TccPatternEnum pattern = tcc.pattern();

       tccTransactionManager.getCurrentTransaction().setPattern(pattern.getCode());


       TccInvocation confirmInvocation = null;
       if (StringUtils.isNoneBlank(confirmMethodName)) {
           confirmInvocation = new TccInvocation(clazz,
                   confirmMethodName, method.getParameterTypes(), args);
       }

       TccInvocation cancelInvocation = null;
       if (StringUtils.isNoneBlank(cancelMethodName)) {
           cancelInvocation = new TccInvocation(clazz,
                   cancelMethodName,
                   method.getParameterTypes(), args);
       }


       //封装调用点
       final Participant participant = new Participant(
               transId,
               confirmInvocation,
               cancelInvocation);

       tccTransactionManager.enlistParticipant(participant);

   }

```
* 这里获取真实的confrim，cancel方法并存入当前的事务信息中。然后发起真实的业务调用 ，即执行payment方法：

```java

  @Override
  @Tcc(confirmMethod = "confirm", cancelMethod = "cancel")
  public boolean payment(AccountDTO accountDTO) {
      final AccountDO accountDO = accountMapper.findByUserId(accountDTO.getUserId());
      accountDO.setBalance(accountDO.getBalance().subtract(accountDTO.getAmount()));
      accountDO.setFreezeAmount(accountDO.getFreezeAmount().add(accountDTO.getAmount()));
      accountDO.setUpdateTime(new Date());
      final int update = accountMapper.update(accountDO);
      if (update != 1) {
          throw new TccRuntimeException("资金不足！");
      }
      return Boolean.TRUE;
  }

````

* 当我们执行完该方法后，会返回，还记得我是在哪里来执行这个方法的吗？对，当然是切面，我们是在切面里执行的，我们是在
**PaymentServiceImpl.makePayment** 切面里面执行的！ 请要理解这一点，执行完后，我们发起了 **inventoryService.decrease(inventoryDTO)** 调用
他的调用原理和上面一模一样，只是在不同的模块里面执行。当 **makePayment** 方法执行完后，我们该怎么执行？ 你还记得 StartTccTransactionHandler吗，它可一直在那等呢。。
我们再来回顾下他的代码:

``` java
@Override
   public Object handler(ProceedingJoinPoint point, TccTransactionContext context) throws Throwable {
       Object returnValue;
       try {
           tccTransactionManager.begin();
           try {
               //发起调用 执行try方法
               returnValue = point.proceed();

           } catch (Throwable throwable) {
               //异常执行cancel

               tccTransactionManager.cancel();

               throw throwable;
           }
           //try成功执行confirm confirm 失败的话，那就只能走本地补偿
           tccTransactionManager.confirm();
       } finally {
           tccTransactionManager.remove();
       }
       return returnValue;
   }
```
* 说到底，我们走了这么久，其实到这里，我们才执行完 **returnValue = point.proceed();** 这一句代码。

#### 没有异常
* 我们会执行 **tccTransactionManager.confirm();**  我们跟进去看代码:

``` java
/**
     * 调用confirm方法 这里主要如果是发起者调用 这里调用远端的还是原来的方法，不过上下文设置了调用confirm
     * 那么远端的服务则会调用confirm方法。。
     */
    void confirm() throws TccRuntimeException {

        LogUtil.debug(LOGGER, () -> "开始执行tcc confirm 方法！start");

        final TccTransaction currentTransaction = getCurrentTransaction();

        if (Objects.isNull(currentTransaction)) {
            return;
        }

        currentTransaction.setStatus(TccActionEnum.CONFIRMING.getCode());

        coordinatorCommand.execute(new CoordinatorAction(CoordinatorActionEnum.UPDATE, currentTransaction));

        final List<Participant> participants = currentTransaction.getParticipants();
        List<Participant> participantList = Lists.newArrayListWithCapacity(participants.size());
        boolean success = true;
        Participant fail = null;
        if (CollectionUtils.isNotEmpty(participants)) {
            for (Participant participant : participants) {
                try {
                    TccTransactionContext context = new TccTransactionContext();
                    context.setAction(TccActionEnum.CONFIRMING.getCode());
                    context.setTransId(participant.getTransId());
                    TransactionContextLocal.getInstance().set(context);
                    //通过反射调用rpc的confrim方法
                    executeParticipantMethod(participant.getConfirmTccInvocation());
                    participantList.add(participant);
                } catch (Exception e) {
                    LogUtil.error(LOGGER, "执行confirm方法异常:{}", () -> e);
                    success = false;
                    fail = participant;
                    break;
                }
            }
        }
        executeHandler(success, currentTransaction, fail, participantList, participants);
    }
    private void executeHandler(boolean success, final TccTransaction currentTransaction, Participant fail,
                                List<Participant> participantList, final List<Participant> participants) {
        if (success) {
            TransactionContextLocal.getInstance().remove();
            coordinatorCommand.execute(new CoordinatorAction(CoordinatorActionEnum.DELETE, currentTransaction));
        } else {
            //获取还没执行的，或者执行失败的
            final List<Participant> updateList =
                    participants.stream().skip(participantList.size()).collect(Collectors.toList());
            currentTransaction.setParticipants(updateList);
            coordinatorCommand.execute(new CoordinatorAction(CoordinatorActionEnum.UPDATE, currentTransaction));
            assert fail != null;
            throw new TccRuntimeException(fail.getConfirmTccInvocation().toString());
        }
    }

    private void executeParticipantMethod(TccInvocation tccInvocation) throws Exception {
       if (Objects.nonNull(tccInvocation)) {
           final Class clazz = tccInvocation.getTargetClass();
           final String method = tccInvocation.getMethodName();
           final Object[] args = tccInvocation.getArgs();
           final Class[] parameterTypes = tccInvocation.getParameterTypes();
           final Object bean = SpringBeanUtils.getInstance().getBean(clazz);
           MethodUtils.invokeMethod(bean, method, args, parameterTypes);

       }
   }

```

* 这段代码的逻辑，简单理解起来，首先更新当前事务状态（confrim），获取当前事务的调用点的confrim方法，设置上下文，发起反射调用！

* 其实这里通过调试我们发现，发起confrim的方法为 **AccountService.payment(AccountDTO accountDTO)**  ，不过设置的上下文状态为confrim,
当我们发起反射调用的时候，我们会走到 **ProviderTccTransactionHandler.handler** 方法,这个方法或许你还有印象，我们再看一下它的代码:

```java
@Override
    public Object handler(ProceedingJoinPoint point, TccTransactionContext context) throws Throwable {
        TccTransaction tccTransaction = null;
        try {
            switch (TccActionEnum.acquireByCode(context.getAction())) {
                case TRYING:
                    try {
                        //创建事务信息
                        tccTransaction = tccTransactionManager.providerBegin(context);
                        //发起方法调用
                        return point.proceed();
                    } catch (Throwable throwable) {
                        tccTransactionManager.removeTccTransaction(tccTransaction);
                        throw throwable;

                    }
                case CONFIRMING:
                    //如果是confirm 通过之前保存的事务信息 进行反射调用
                    final TccTransaction acquire = tccTransactionManager.acquire(context);
                    tccTransactionManager.confirm();
                    break;
                case CANCELING:
                    //如果是调用CANCELING 通过之前保存的事务信息 进行反射调用
                    tccTransactionManager.acquire(context);
                    tccTransactionManager.cancel();
                    break;
                default:
                    break;
            }
        } finally {
            tccTransactionManager.remove();
        }
        Method method = ((MethodSignature) (point.getSignature())).getMethod();
        return getDefaultValue(method.getReturnType());
    }

```
* 这里因为上下文设置的状态为:CONFIRMING ,所以会执行:

```java
//如果是confirm 通过之前保存的事务信息 进行反射调用
final TccTransaction acquire = tccTransactionManager.acquire(context);
tccTransactionManager.confirm();
break;
```
* 我们跟踪 **tccTransactionManager.confirm();** 会发现和之前是一个方法，这时候，你要知道，这个方法是在account微服务里面执行

* 所以它最后会执行 **AccountServiceImpl.confirm** 方法，进行了付款确认。


### 同理cancel方法也和上面描述的一样的原理执行。


## 到这里，我们就解析完了，整个tcc过程执行的流程，大家关键要理解AOP，理解好了切面思想，其实是很简单的事情了，如果有任何疑问和问题，欢迎加入QQ群：162614487 进行讨论
