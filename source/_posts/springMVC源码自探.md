title: springMVC 源码自探
date: 2019-01-17 15:28:30
tags: [源码, springMVC]
categories: [springMVC]
---
## springMVC 源码自探

![I love it when a plan comes together.](http://ww1.sinaimg.cn/large/006Cwrd9gy1fxskn2tpksj31hc0u0guq.jpg)
### 前言
弄清楚springMVC流程对我们写rest接口有莫大的好处，因为和它绑定在一起的还有这些重要的东西：
> 1. http/https的知识。http请求的组成。
2. 消息转化(参数是**如何解析**的，**什么时候解析**的，我们给常用的注解是如何实现的)
3. springMVC拦截器的实现原理
4. 其他，如消息监控的植入actuator,zipkin等。

### spring的启动
>spring启动的时候会扫描各个controller，并默认以单例的形式生成各个handler(controller类),handlerMethod().

### 一个rest请求的到来
#### 入口
> 如果入口是从最开始说的话其实是从tomcat的各个过滤器(filter)顺序调用过来的，再由httpServlet拉起springMVC框架的service方法
确定方法类型后执行doPost，或者doGet。下面只会会将几个重要过程领出来聊一聊

#### DispatcherServlet.doDispatch()
>先从10个handlerMappings中的找到对应的handlerMapping,然后从这个handlerMapping通过urlPath，找到对应方法的handler
 并将拦截器加入handler组装成HandlerExecutionChain，如果执行HandlerExecutionChain的时候拦截器返回false方法会在止到这里直接返回。
 拦截器如果通过，找到能够使用这个handler(Method Handler)的handlerAdapter（一般是这个:RequestMappingHandlerAdapter）
 （原本有三个找到对应的支持的） 供以后使用，也就是下面的这个方法：
#### ServletInvocableHandlerMethod.invokeAndHandle()
>执行里面的方法，并处理返回值
 a. 设置响应状态，
 b. 设置mavContainer处理状态设置为未处理完毕。
 c. 并对返回接口进行处理,responseBody的注解会在这里使用(用于找到返回值的handler)，并进行消息转化(json)
#### InvocableHandlerMethod.invokeForRequest() 
>这里会解析并映射入参，这里会有各种解析参数的解析器，找到对应的解析器后然后会根据参数名字到request里面取值。
后放入到对应的参数列表里面再将参数放入代理方法执行invoke，并返回业务代码里的返回值。
springMVC的字段名在request里面是这样的：如果是get方法一般就是上面的parameterMap里面的。
 