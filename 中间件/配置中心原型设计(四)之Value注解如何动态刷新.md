



`com.ctrip.framework.apollo.spring.annotation.SpringValueProcessor`是Apollo中对于@Value注解动态值刷新的实现。
该类继承了 `ApolloProcessor` ,然后 `ApolloProcessor` 又是实现了 `BeanPostProcessor` ,所以该类会处理所有的bean实例，对所有的 `Field` 和 `Method` 进行处理。


先看一下`com.ctrip.framework.apollo.spring.annotation.SpringValueProcessor#processField`, 这里会对 `Field` 进行处理

1. `field.getAnnotatio`  >>  对使用了Value注解的进行过滤
2. `placeholderHelper.extractPlaceholderKeys`  >>  提取Value注解中的key值对象
3. `new SpringValue`  >>  包装成对象
4. `springValueRegistry.register`  >> 将所有的数据，根据 `beanFactory` 存入 `Map<BeanFactory, Multimap<String, SpringValue>>` 中。

```java
@Override
protected void processField(Object bean, String beanName, Field field) {
    // register @Value on field
    Value value = field.getAnnotation(Value.class);
    if (value == null) {
      return;
    }
    Set<String> keys = placeholderHelper.extractPlaceholderKeys(value.value());
    if (keys.isEmpty()) {
      return;
    }
    for (String key : keys) {
      SpringValue springValue = new SpringValue(key, value.value(), bean, beanName, field, false);
      springValueRegistry.register(beanFactory, key, springValue);
      logger.debug("Monitoring {}", springValue);
    }
}
```


在 `AutoUpdateConfigChangeListener#onchange` 会处理那些发生变化的配置项，这里最终会通过反射的方式将Filed或者Method中的属性值进行修改。

然后这里的配置变化是由谁发起的，通过不断的回溯，发现最终都是调用 `com.ctrip.framework.apollo.internals.AbstractConfigRepository#trySync` 这个方法。 `trySync` 主要的逻辑是调用 `sync` 方法， 这个方法只是一个抽象，具体的实现类会有好几个，这里我们主要关注`com.ctrip.framework.apollo.internals.RemoteConfigRepository#sync`的实现就行了。

这里需要明确一个关系： `RemoteConfigRepository` 继承了 `AbstractConfigRepository`, 并且实现了`AbstractConfigRepository` 中的抽象方法 `sync`.

`RemoteConfigRepository` 调用 `trySync` 方法的地方 主要有三个：

1. `RemoteConfigRepository#schedulePeriodicRefresh`  >>>  这是一个定时任务，每隔5分钟拉一份配置中心
2. `RemoteConfigRepository#onLongPollNotified`  >>>  `RemoteConfigLongPollService`中有长轮询的实现,通过 `while`循环实现的。
3. `RemoteConfigRepository#RemoteConfigRepository`  >>>  构造方法会执行一次





## 关于`BeanPostProcessor` 和 `BeanFactoryPostProcessor`的区别

> https://www.cnblogs.com/dreampig/p/9036077.html

`BeanPostProcessor`: bean级别的处理，针对某个具体的bean进行处理
`BeanFactoryPostProcessor`：BeanFactory级别的处理，是针对整个Bean的工厂进行处理

以上两种都为Spring提供的后处理bean的接口，只是两者执行的时机不一样。`BeanPostProcessor`为实例化之后，`BeanFactoryPostProcessor`是实例化之前。功能上，`BeanFactoryPostProcessor`对bean的处理功能更加强大。