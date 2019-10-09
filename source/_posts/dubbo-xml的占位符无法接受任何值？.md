---
title: dubbo.xml的占位符无法接受任何值？
date: 2019-09-25 20:05:59
tags: [springboot, dubbo]
---

### 背景
最近负责改造公司的老项目到新的技术栈springCloud(springboot2.0X)的时候遇到了一个特别尴尬的问题，就是dubbo的配置文件无法使用占位符来注入,这样会导致无论测试还是生产，每次发版的时候都需要改一下，很显然，这样特别不优雅。

### 探索
终于等到项目整合的末期只剩这个不算bug的bug了，我下定决心打算解决它，于是探索之旅开始了，先是一番老操作：一顿百度google，结果是：只看到有相同提问的却没有一个解答的。(百度出来也不会写博客了)

### 老项目的占位符是这样生效的:
随着源码看一下老项目的占位符生效过程，项目启动的时候扫描先扫描xml配置文件，并调用loadBeanDefinitions，后注册beanDefinition：
![](/images/loadBeanDefinitions.png)
在AbstractApplicationContext#refresh中的invokeBeanFactoryPostProcessors会调用PropertyPlaceholderConfigurer的postProcessBeanFactory:
![](/images/invokeBeanFacotryPostProcessors.png)
postProcessBeanFactory方法对占位符进行处理：processProperties将所有beanDefinition的占位符进行替换(具体的就不讲了,就是遍历beanDefinition,逐个访问替换)
![](/images/processProperties.png)

### PropertyPlaceholderConfigurer
下面是PropertyPlaceholderConfigurer这个类的uml图：
![](/images/PropertyPlaceholderConfigurer.png) 如果我们配置了properties文件等等的最终是以这个bean的形式注入到容器的。 然后在bean实例化之前refresh方法会调用对应的方法使得占位符生效。

### springCloud(springCloud)项目启动是这样生效的:
springboot官方文档里面有这样一段话：
>Caution
While using @PropertySource on your @SpringBootApplication may seem to be a
convenient and easy way to load a custom resource in the Environment, we do not recommend
it, because Spring Boot prepares the Environment before the ApplicationContext is
refreshed. Any key defined with @PropertySource is loaded too late to have any effect on auto configuration.

意思就是说@propertySource 注解起作用的时候太晚了，不能对auto 
configuration起到作用。bean会在在我们还没有解析占位符的时候就初始化，导致启动异常。

**springCloud的启动流程:**
1. 先获取注册中心的注册表
2. 请求configServer,拉回配置信息
3. 由RefreshAutoConfiguration的内部类RefreshScopeBeanDefinitionEnhancer进行刷新配置,这个过程中获要工具类型获取Environment,调用getBean(Environment.class);
4. 通过类型获取一个bean会导致遍历bean,这个过程会调用到isTypeMatch
5. 当遍历到dubbo服务的bean(因为dubbo的service都是FactoryBean)时候的doCreateBean需要依赖RegistryConfig,
6. dubbo里面的工具类获取:BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);而这个时候PropertyPlaceholderConfigurer还没有被调用,导致占位符没有解析连接不上。   

### 解决
知道原因解决起来就相对简单了，在他调用刷新RefreshScopeBeanDefinitionEnhancer之前先进行占位符解析，可以用spring优先级高的扩展点：
~~~java
/**
 * Extension to the standard {@link BeanFactoryPostProcessor} SPI, allowing for
 * the registration of further bean definitions <i>before</i> regular
 * BeanFactoryPostProcessor detection kicks in. In particular,
 * BeanDefinitionRegistryPostProcessor may register further bean definitions
 * which in turn define BeanFactoryPostProcessor instances.
 *
 * @author Juergen Hoeller
 * @since 3.0.1
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	// 这里就是最关键的地方了，这个方法会在bean definition初始化完成之后，但是还没有任何bean被初始化(initializbean)
	// 这个方法允许我们对已经注册了的所有bean definition进行额外的操作。
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
~~~
### 对应代码：
相面两个都可以解决问题，只要实现其中一个就可以了。
~~~ java
@Component
public class DubboRegistryOverride implements BeanDefinitionRegistryPostProcessor, EnvironmentAware {

    private static Environment ENV;

    @Override
    public void setEnvironment(Environment environment) {
        ENV = environment;
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        BeanDefinition registryConfig = registry.getBeanDefinition("com.alibaba.dubbo.config.RegistryConfig");
        // 这里直接取到yml里面的properties的值即可
        String address = ENV.getProperty("dubbo.address");
        String group = ENV.getProperty("dubbo.group");
        MutablePropertyValues propertyValues = registryConfig.getPropertyValues();
        // 覆盖对应key的值
        propertyValues.add("address", address);
        propertyValues.add("group", group);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 或者使用BeanFactoryPostProcessor对应的beanFactory里面的值
        RegistryConfig registryConfig = beanFactory.getBean(RegistryConfig.class);
        // 直接获取到对应的dubbo配置即可
        registryConfig.setAddress(ENV.getProperty("dubbo.group"));
        registryConfig.setGroup(ENV.getProperty("dubbo.group"));
    }
}
~~~

### 回顾
解决这个问题其实走了挺多弯路的，特别是最开始百度的时候，没什么思路，随着对问题的一步一步的探究，跟着出问题的源码，在加上对源码的一部分技艺，也找到了解答的方向，最终找到了答案。
