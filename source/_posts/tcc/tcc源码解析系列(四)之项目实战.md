---
title: tcc源码解析系列(四)之项目实战
date: 2017-10-12 18:03:53
categories: hmily-tcc
permalink: TCC/tcc-four
---

#### 通过之前的几篇文章我相信您已经搭建好了运行环境，本次的项目实战是依照[hmily-tcc-demo](https://github.com/yu199195/hmily/tree/master/hmily-tcc-demo)项目来演练，也是非常经典的分布式事务场景：支付成功，进行订单状态的更新，扣除用户账户，库存扣减这几个模块来进行tcc分布式事务。话不多说，让我们一起进入体验吧！


* 首先我们找到 **PaymentServiceImpl.makePayment** 方法，这是tcc分布式事务的发起者

```java
   //tcc分布式事务的发起者
   @Tcc(confirmMethod = "confirmOrderStatus", cancelMethod = "cancelOrderStatus")
   public void makePayment(Order order) {
       order.setStatus(OrderStatusEnum.PAYING.getCode());
       orderMapper.update(order);
       //扣除用户余额 远端的rpc调用
       AccountDTO accountDTO = new AccountDTO();
       accountDTO.setAmount(order.getTotalAmount());
       accountDTO.setUserId(order.getUserId());
       accountService.payment(accountDTO);
       //进入扣减库存操作 远端的rpc调用
       InventoryDTO inventoryDTO = new InventoryDTO();
       inventoryDTO.setCount(order.getCount());
       inventoryDTO.setProductId(order.getProductId());
       inventoryService.decrease(inventoryDTO);
   }
   //更新订单的confirm方法
   public void confirmOrderStatus(Order order) {
        order.setStatus(OrderStatusEnum.PAY_SUCCESS.getCode());
        orderMapper.update(order);
        LOGGER.info("=========进行订单confirm操作完成================");


    }
    //更新订单的cancel方法
    public void cancelOrderStatus(Order order) {
        order.setStatus(OrderStatusEnum.PAY_FAIL.getCode());
        orderMapper.update(order);
        LOGGER.info("=========进行订单cancel操作完成================");
    }

```
* makePayment 方法是tcc分布式事务的发起者，它里面有更新订单状态（本地），进行库存扣减（rpc扣减），资金扣减(rpc调用)

* **confirmOrderStatus** 为订单状态confirm方法，**cancelOrderStatus** 为订单状态cancel方法。

* 我们重点关注 **@Tcc(confirmMethod = "confirmOrderStatus", cancelMethod = "cancelOrderStatus")** 这个注解，为什么加了这个注解就有神奇的作用。

### @Tcc注解切面

* 我们找到在[hmily-tcc-core](https://github.com/yu199195/hmily/tree/master/hmily-tcc-core)包中找到
 **TccTransactionAspect**，这里定义@Tcc的切点。

```java
@Aspect
public abstract class TccTransactionAspect {

    private TccTransactionInterceptor tccTransactionInterceptor;

    public void setTccTransactionInterceptor(TccTransactionInterceptor tccTransactionInterceptor) {
        this.tccTransactionInterceptor = tccTransactionInterceptor;
    }

    @Pointcut("@annotation(com.hmily.tcc.annotation.Tcc)")
    public void txTransactionInterceptor() {

    }

    @Around("txTransactionInterceptor()")
    public Object interceptCompensableMethod(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        return tccTransactionInterceptor.interceptor(proceedingJoinPoint);
    }

    public abstract int getOrder();
}

```
* 我们可以知道Spring实现类的方法凡是加了@Tcc注解的，在调用的时候，都会进行 **tccTransactionInterceptor.interceptor** 调用。

* 其次我们发现该类是一个抽象类，肯定会有其他类继承它，你猜的没错，对应dubbo用户，他的继承类为:

``` java
@Aspect
@Component
public class DubboTccTransactionAspect extends TccTransactionAspect implements Ordered {


    @Autowired
    public DubboTccTransactionAspect(DubboTccTransactionInterceptor dubboTccTransactionInterceptor) {
        super.setTccTransactionInterceptor(dubboTccTransactionInterceptor);
    }


    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}

```

* springcloud的用户,它的继承类为:

``` java
@Aspect
@Component
public class SpringCloudTccTransactionAspect extends TccTransactionAspect implements Ordered {


    @Autowired
    public SpringCloudTxTransactionAspect(SpringCloudTccTransactionAspect  springCloudTccTransactionAspect) {
        this.setTccTransactionInterceptor(springCloudTccTransactionAspect);
    }


    public void init() {

    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}

```

* 我们注意到他们都实现了Spring的Ordered接口，并重写了 **getOrder** 方法，都返回了 **Ordered.HIGHEST_PRECEDENCE** 那么可以知道，他是优先级最高的切面。

###  TccCoordinatorMethodAspect 切面
```java
@Aspect
@Component
public class TccCoordinatorMethodAspect implements Ordered {


    private final TccCoordinatorMethodInterceptor tccCoordinatorMethodInterceptor;

    @Autowired
    public TccCoordinatorMethodAspect(TccCoordinatorMethodInterceptor tccCoordinatorMethodInterceptor) {
        this.tccCoordinatorMethodInterceptor = tccCoordinatorMethodInterceptor;
    }


    @Pointcut("@annotation(com.hmily.tcc.annotation.Tcc)")
    public void coordinatorMethod() {

    }

    @Around("coordinatorMethod()")
    public Object interceptCompensableMethod(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        return tccCoordinatorMethodInterceptor.interceptor(proceedingJoinPoint);
    }


    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }
}

```

* 该切面是第二优先级的切面，意思就是当我们对有@Tcc注解的方法发起调用的时候，首先会进入 **TccTransactionAspect** 切面，然后再进入 **TccCoordinatorMethodAspect** 切面。
该切面主要是用来获取@Tcc注解上的元数据，比如confrim方法名称等等。

* 到这里如果大家基本能明白的话，差不多就了解了整个框架原理，现在让我们来跟踪调用吧。


### 现在我们回过头，我们来调用 **PaymentServiceImpl.makePayment** 方法

* dubbo用户，我们会进入如下拦截器:

```java

@Component
public class DubboTccTransactionInterceptor implements TccTransactionInterceptor {

    private final TccTransactionAspectService tccTransactionAspectService;

    @Autowired
    public DubboTccTransactionInterceptor(TccTransactionAspectService tccTransactionAspectService) {
        this.tccTransactionAspectService = tccTransactionAspectService;
    }


    @Override
    public Object interceptor(ProceedingJoinPoint pjp) throws Throwable {
        final String context = RpcContext.getContext().getAttachment(Constant.TCC_TRANSACTION_CONTEXT);
        TccTransactionContext tccTransactionContext = null;
        if (StringUtils.isNoneBlank(context)) {
            tccTransactionContext =
                    GsonUtils.getInstance().fromJson(context, TccTransactionContext.class);
        }
        return tccTransactionAspectService.invoke(tccTransactionContext, pjp);
    }
}

```
* 我们继续跟踪 **tccTransactionAspectService.invoke(tccTransactionContext, pjp)** ,发现通过判断过后，我们会进入 **StartTccTransactionHandler.handler** 方法,我们已经进入了分布式事务处理的入口了！

``` java
/**
    * 分布式事务处理接口
    *
    * @param point   point 切点
    * @param context 信息
    * @return Object
    * @throws Throwable 异常
    */
   @Override
   public Object handler(ProceedingJoinPoint point, TccTransactionContext context) throws Throwable {
       Object returnValue;
       try {
          //开启分布式事务
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
* 我们进行跟进 **tccTransactionManager.begin()** 方法:

```java
/**
    * 该方法为发起方第一次调用
    * 也是tcc事务的入口
    */
   void begin() {
       LogUtil.debug(LOGGER, () -> "开始执行tcc事务！start");
       TccTransaction tccTransaction = CURRENT.get();
       if (Objects.isNull(tccTransaction)) {
           tccTransaction = new TccTransaction();
           tccTransaction.setStatus(TccActionEnum.TRYING.getCode());
           tccTransaction.setRole(TccRoleEnum.START.getCode());
       }
       //保存当前事务信息
       coordinatorCommand.execute(new CoordinatorAction(CoordinatorActionEnum.SAVE, tccTransaction));

       CURRENT.set(tccTransaction);

       //设置tcc事务上下文，这个类会传递给远端
       TccTransactionContext context = new TccTransactionContext();
       context.setAction(TccActionEnum.TRYING.getCode());//设置执行动作为try
       context.setTransId(tccTransaction.getTransId());//设置事务id
       TransactionContextLocal.getInstance().set(context);

   }
```
* 这里我们保存了事务信息，并且开启了事务上下文，并把它保存在了ThreadLoacl里面，大家想想这里为什么一定要保存在ThreadLocal里面。

* begin方法执行完后，我们回到切面，现在我们来执行 **point.proceed()**，当执行这一句代码的时候，会进入第二个切面，即进入了 **TccCoordinatorMethodInterceptor**

```java

public Object interceptor(ProceedingJoinPoint pjp) throws Throwable {

       final TccTransaction currentTransaction = tccTransactionManager.getCurrentTransaction();

       if (Objects.nonNull(currentTransaction)) {
           final TccActionEnum action = TccActionEnum.acquireByCode(currentTransaction.getStatus());
           switch (action) {
               case TRYING:
                   registerParticipant(pjp, currentTransaction.getTransId());
                   break;
               case CONFIRMING:
                   break;
               case CANCELING:
                   break;
           }
       }
       return pjp.proceed(pjp.getArgs());
   }

```

* 这里由于是在try阶段，直接就进入了 **registerParticipant** 方法

```java
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

* 这里就获取了 **@Tcc(confirmMethod = "confirmOrderStatus", cancelMethod = "cancelOrderStatus")**
的信息，并把他封装成了 Participant，存起来，然后真正的调用 **PaymentServiceImpl.makePayment** 业务方法，在业务方法里面，我们首先执行的是更新订单状态（相信现在你还记得0.0）:

``` java
       order.setStatus(OrderStatusEnum.PAYING.getCode());
       orderMapper.update(order);
```

* 接下来，我们进行扣除用户余额,注意扣除余额这里是一个rpc方法：

``` java
       AccountDTO accountDTO = new AccountDTO();
       accountDTO.setAmount(order.getTotalAmount());
       accountDTO.setUserId(order.getUserId());
       accountService.payment(accountDTO)
```

* 现在我们来关注 **accountService.payment(accountDTO)** ，这个接口的定义。

#### dubbo接口:

```java

public interface AccountService {


    /**
     * 扣款支付
     *
     * @param accountDTO 参数dto
     * @return true
     */
    @Tcc
    boolean payment(AccountDTO accountDTO);
}

```

#### springcloud接口

``` java
@FeignClient(value = "account-service", configuration = MyConfiguration.class)
public interface AccountClient {

    @PostMapping("/account-service/account/payment")
    @Tcc
    Boolean payment(@RequestBody AccountDTO accountDO);

}
```

### 很明显这里我们都在接口上加了@Tcc注解，我们知道springAop的特性，在接口上加注解，是无法进入切面的，所以我们在这里，要采用rpc框架的某些特性来帮助我们获取到 @Tcc注解信息。 这一步很重要。当我们发起
**accountService.payment(accountDTO)** 调用的时候:

* dubbo用户，会走dubbo的filter接口,**TccTransactionFilter**:

```java
private void registerParticipant(Class clazz, String methodName, Object[] arguments, Class... args) throws TccRuntimeException {
      try {
          Method method = clazz.getDeclaredMethod(methodName, args);
          Tcc tcc = method.getAnnotation(Tcc.class);
          if (Objects.nonNull(tcc)) {

            //获取事务的上下文
              final TccTransactionContext tccTransactionContext =
                      TransactionContextLocal.getInstance().get();
              if (Objects.nonNull(tccTransactionContext)) {
                //dubbo rpc传参数
                  RpcContext.getContext()
                          .setAttachment(Constant.TCC_TRANSACTION_CONTEXT,
                                  GsonUtils.getInstance().toJson(tccTransactionContext));
              }
              if (Objects.nonNull(tccTransactionContext)) {
                  if(TccActionEnum.TRYING.getCode()==tccTransactionContext.getAction()){
                      //获取协调方法
                      String confirmMethodName = tcc.confirmMethod();

                      if (StringUtils.isBlank(confirmMethodName)) {
                          confirmMethodName = method.getName();
                      }

                      String cancelMethodName = tcc.cancelMethod();

                      if (StringUtils.isBlank(cancelMethodName)) {
                          cancelMethodName = method.getName();
                      }
                    //设置模式
                      final TccPatternEnum pattern = tcc.pattern();

                      tccTransactionManager.getCurrentTransaction().setPattern(pattern.getCode());
                      TccInvocation confirmInvocation = new TccInvocation(clazz,
                              confirmMethodName,
                              args, arguments);

                      TccInvocation cancelInvocation = new TccInvocation(clazz,
                              cancelMethodName,
                              args, arguments);

                      //封装调用点
                      final Participant participant = new Participant(
                              tccTransactionContext.getTransId(),
                              confirmInvocation,
                              cancelInvocation);

                      tccTransactionManager.enlistParticipant(participant);
                  }

              }



          }
      } catch (NoSuchMethodException e) {
          throw new TccRuntimeException("not fount method " + e.getMessage());
      }

  }
```

* springcloud 用户，则会进入 **TccFeignHandler** ：

```java
public class TccFeignHandler implements InvocationHandler {
    /**
     * logger
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(TccFeignHandler.class);

    private Target<?> target;
    private Map<Method, MethodHandler> handlers;

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else {

            final Tcc tcc = method.getAnnotation(Tcc.class);
            if (Objects.isNull(tcc)) {
                return this.handlers.get(method).invoke(args);
            }

            final TccTransactionContext tccTransactionContext =
                    TransactionContextLocal.getInstance().get();
            if (Objects.nonNull(tccTransactionContext)) {
                final TccTransactionManager tccTransactionManager =
                        SpringBeanUtils.getInstance().getBean(TccTransactionManager.class);
                if (TccActionEnum.TRYING.getCode() == tccTransactionContext.getAction()) {
                    //获取协调方法
                    String confirmMethodName = tcc.confirmMethod();

                    if (StringUtils.isBlank(confirmMethodName)) {
                        confirmMethodName = method.getName();
                    }

                    String cancelMethodName = tcc.cancelMethod();

                    if (StringUtils.isBlank(cancelMethodName)) {
                        cancelMethodName = method.getName();
                    }

                    //设置模式
                    final TccPatternEnum pattern = tcc.pattern();

                    tccTransactionManager.getCurrentTransaction().setPattern(pattern.getCode());

                    final Class<?> declaringClass = method.getDeclaringClass();

                    TccInvocation confirmInvocation = new TccInvocation(declaringClass,
                            confirmMethodName,
                            method.getParameterTypes(), args);

                    TccInvocation cancelInvocation = new TccInvocation(declaringClass,
                            cancelMethodName,
                            method.getParameterTypes(), args);

                    //封装调用点
                    final Participant participant = new Participant(
                            tccTransactionContext.getTransId(),
                            confirmInvocation,
                            cancelInvocation);

                    tccTransactionManager.enlistParticipant(participant);
                }

            }


            return this.handlers.get(method).invoke(args);
        }
    }


    public void setTarget(Target<?> target) {
        this.target = target;
    }


    public void setHandlers(Map<Method, MethodHandler> handlers) {
        this.handlers = handlers;
    }

}
```

### 重要提示 在这里我们获取了在第一步设置的事务上下文，然后进行dubbo的rpc传参数，细心的你可能会发现，这里获取远端confirm，cancel方法其实就是自己本身啊？，那发起者还怎么来调用远端的confrim，cancel方法呢？这里的调用，我们通过事务上下文的状态来控制，比如在confrim阶段，我们就调用confrim方法等。。

## 到这里我们就完成了整个消费端的调用，下一篇分析提供者的调用流程！
