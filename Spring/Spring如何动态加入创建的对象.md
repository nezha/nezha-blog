# `Spring`如何动态加入创建的对象

## 1.如果将`new`的对象交给`Spring`管理

- 1.新建一个配置类
```Java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        for (int i = 0; i < 10; i++) {
            BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(Person.class);
            builder.addPropertyValue("name", "sfz_" + i);
            beanDefinitionRegistry.registerBeanDefinition("person" + i, builder.getBeanDefinition());
        }
        beanDefinitionRegistry.removeBeanDefinition("person5");
        beanDefinitionRegistry.containsBeanDefinition("xxxx");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}
```

- 2.启动程序验证

```java
@SpringBootApplication
public class EurekaServerAApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(EurekaServerAApplication.class, args);
        context.getBeansOfType(Person.class).values().forEach(System.out::println);
    }
}
```


## 参考文献



1.  