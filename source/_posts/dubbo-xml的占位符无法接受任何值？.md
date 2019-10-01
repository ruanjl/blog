---
title: dubbo.xml的占位符无法接受任何值？
date: 2019-09-25 20:05:59
tags: [springboot, dubbo]
---

### 背景
最近负责改造公司的老项目到新的技术栈(springboot2.0X)的时候遇到了一个特别尴尬的问题，就是dubbo的配置文件无法使用占位符来注入,这样会导致无论测试还是生产，每次发版的时候都需要改一下，很显然，这样特别不优雅。

### 探索
终于等到项目整合的末期只剩这个不算bug的bug了，我下定决心打算解决它，于是探索之旅开始了，先是一番老操作：一顿百度google，结果是：只看到有相同提问的却没有一个解答的。 (百度出来也不会写博客了)

### springFramework3.1和springboot的对于占位符解析的区别.
由于最近也在看spring源码这一块的东西，刚好用上了，老系统springFramework在启动的时候是先读取xml配置文件, 接着读取properties文件，并解析占位符，然后才会开始注册beanDefinitions,在实例化之前属性其实就已经解析完了。 实例化接着初始化复制后连接注册中心,然后开始的其他操作。 而springboot官方文档里面有这样一段话：
>Caution
While using @PropertySource on your @SpringBootApplication may seem to be a
convenient and easy way to load a custom resource in the Environment, we do not recommend
it, because Spring Boot prepares the Environment before the ApplicationContext is
refreshed. Any key defined with @PropertySource is loaded too late to have any effect on auto configuration.

**也就是说@propertySource 注解起作用的时候太晚了**，不能对auto configuration 起到作用。 我也想通过在yml里面写对应的属性值，但是我发现当springboot去加载xml bean的parse后registry的时候并不会对占位符进行解析 也就是beanDefinition里面就还是原来的占位符的字符串：而springFramework同一个时候是已经解析了的 好在spring可以在beanDefinition实例化之前支持扩展beanDefinition也就是实现接口 BeanDefinitionRegistryPostProcessor;这里有两个方法,我们需要实现的是这个接口：

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

### 解决
spring的bean在真正的初始化之前有两个重要的扩展点,使得框架更加灵活,通过实现指定的扩展点接口我们可以修改bean实例化之前(beanDefinition, beanFactory)的对应的属性值，从而来改变实例化的时候属性的值:
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
解决这个问题其实走了挺多弯路的，特别是最开始百度的时候，没什么思路，随着对问题的一步一步的探究，也找到了解答的方向，最终找到了答案。
