---
title: springboot线程池与@Async
date: 2020-01-09 12:31:31
tags: [springboot, async]
---
## 前言
公司的很多老项目的线程管理十分混乱，最近这里的一些springboot的这一块的东西，最新的springboot 2.2.2已经将线程池的配置直接纳入自动配置了
[文档地址](https://docs.spring.io/spring-boot/docs/2.2.2.RELEASE/reference/html/spring-boot-features.html#boot-features-task-execution-scheduling)
虽然我们现在版本的springboot还没有这个功能，但是我们也应该类似的将线程池的管理起来：

## @Async注解
springboot 使用线程池的时候用@Async是比较优雅的方式，因为这样不用显示注入线程池，而是bean的时候使用代理实现的，对我们是无感知的。  
使用的时候开启@EnableAsync并在对应的异步方法上添加@Async即可，spring就会使用配置好的对应的线程池。  
注意：让spring管理我们的线程池需要符合一定的要求：
> By default, Spring will be searching for an associated thread pool definition: either a unique {@link
> org.springframework.core.task.TaskExecutor} bean in the context, or an {@link java.util.concurrent.Executor} bean
> named "taskExecutor" otherwise. If neither of the two is resolvable, a {@link
> org.springframework.core.task.SimpleAsyncTaskExecutor} will be used to process async method invocations. Besides,
> annotated methods having a {@code void} return type cannot transmit any exception back to the caller. By default, such
> uncaught exceptions are only logged.

也就是说需要满足下面这两点其中一个
1.  声明一个唯一的TaskExecutor,
2. 或者声明一个beanName是taskExecutor的Executor。

## @Async使用注意事项
@Async的执行的使用分两种情况：
1. 没有返回值的(void):
- 这种可以直接使用即可。
2. 有返回值:
- 有返回值的情况需要返回值类型必须得是java.util.concurrent.Future类型的，获取真真的结果的时候调用Future.get()来获取对应的结果。
- 这种情况下，可以声明实现了java.util.concurrent.Future的更具体的返回值类型:
- org.springframework.util.concurrent.ListenableFuture或java.util.concurrent.CompletableFuture类型，
- 这两种类型可以让我们与异步任务有更丰富的交互和或者进一步的处理异步结果。
- CompletableFuture类型有一些需要注意的需要在另外的一篇详细讲解一下。

## 合理的配置线程池
线程池的核心参数主要如下：
1. 核心线程
2. 最大线程数
3. 队列大小
这些参数的配置主要和项目的任务类型和机器配置相关,网络上百度有很多资料，这里主要说两点注意事项：
1. 拒绝任务的时候一定要打印相关的日志，无论这个任务是否是重要的，例如下面的例子，并用一个原子变量来监控任务拒绝情况。
2. 使用@Async注解来对应的异步任务的时候，需要将本任务的执行情况写在对应的方法上面。比如多少时间内完成多少任务。并给出每个任务大致的消耗时间。

**具体可以参考如下的配置，线程池的线程数等等仅供参考。和自己的机器和业务相关性大。**
```
@EnableAsync
@Configuration
public class IaskAdThreadPool {

    /**
     * rejectCount 监控任务繁忙情况
     */
    private AtomicLong rejectCount = new AtomicLong();

    /**
     * iask线程池
     *
     * @return
     */
    @Bean
    public TaskExecutor taskExecutor(){
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setThreadNamePrefix("iask-ad-task");
        taskExecutor.setCorePoolSize(64);
        taskExecutor.setMaxPoolSize(256);
        taskExecutor.setQueueCapacity(2048);
        taskExecutor.setKeepAliveSeconds(60 * 2);
        taskExecutor.setRejectedExecutionHandler((r, executor) -> {
            log.info("提交任务{},时任务队列已满, 已调用CallerRunsPolicy{}次", r.toString(), rejectCount.incrementAndGet());
            new ThreadPoolExecutor.CallerRunsPolicy().rejectedExecution(r, executor);
        });
        return taskExecutor;
    }
}
```
