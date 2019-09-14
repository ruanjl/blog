---
title: springFramework3.1这样的老版本怎么维护啊？
date: 2019-09-05 23:08:04
tags: [springFramework3.1, rabbitMq]
---

### 能活下来的老项目是宝
我们不可能一直是使用最新的技术，是项目总会成为过去，而项目能活下来也是不容易的,都是有他的价值的，而这个时候如果让你顶上去维护老项目你能行么。

### 起因
最近由于技术栈的变更, 也就是从activeMq转变为rabbitMQ这一块，项目先是出了一个bug，是有关延时队列的使用：由于rabbiMq的延时队列是使用死信队列实现的，而这个队列是一个单向阻塞队列，如果一个延时长的任务先入队，那么后面延时短的任务要等到前面的元素出队列以后才能继续出队。这样会造成后面消息无法按时出队,于是网上搜索了下解决方案，很快就搜索到了，使用一个rabbitMq延时插件可以实现想要的功能，但是问题却也随之来了

### springFramework3.1老项目
老项目的配置都是使用xml维护的，而网上的解决方案都是基于springboot的解决方案，spring Framework的解决延时队列的方案我是没有找到 于是我只好将对应配置的配置项的xml标签都看了一遍，根本就没有对应解决方案的标签CustomExchange，于是我想着使用@bean 标签去注入，结果发现无法注入， 因为这个spring的版本太低了，于是到[官方文档](https://docs.spring.io/spring/docs/3.1.1.RELEASE/spring-framework-reference/html/) 查看了下相关的资料，发现这个版本是刚刚出 @bean 这个注解的时候，这个时候的这个注解功能还处于特别低级的情况。 因为官方文档上面写的基本都还是在使用xml配置。这个时候是我们熟知的@bean @configuration注解刚刚出道的时候。

### 无法注入
我按照官网的方法使用@configuration 和@bean 去注入，都没法注入bean，启动的时候一直报无法找到factory method异常导致的无法启动，
于是我开始仔细看这两个注解目前版本的的功能究竟是什么，结果是这样的：
> @configuration注解基本没什么用处,因为你要使用的话的需要用注解扫描的方式启动项目才可以

直到后来我将注意力转移到@bean的使用的时候看到一句话：  
> 4.12.4 Using the @Bean annotation
@Bean is a method-level annotation and a direct analog of the XML <bean/> element. 
The annotation supports some of the attributes offered by <bean/>, such as: init-method, destroy-method, autowiring and name.
You can use the @Bean annotation in a @Configuration-annotated or in a @Component-annotated class.  

答案在最后面一句上：你也可以使用@Component注解的java类上使用@bean注解，
我这样做了之后还是没有注入我想要的自定义的bean，于是我推测既然是基于xml扫描的配置启动是不是还要在xml里面声明一下这个java类，
声明之后终于可以按预期run起来了。而且可以完好的依赖成功后运行处想要的效果，到此，终于解决了这个问题。

### 总结
虽然最后的解决方案，看上去还是比较简单的，但是总不可能凭空产生这样的结果，当我们网上搜索不到答案的时候，也要冷静来认真分析问题，
不要仅限于当前springboot的如此方便的技术，还要知道对应原理，历史来源，这样才不至于遇到baidu或者google不到的问题就放弃了。



