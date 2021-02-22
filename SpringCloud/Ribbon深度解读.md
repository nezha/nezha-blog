# Ribbon LoadBalance深度解读

Ribbon在微服务体系中的重要性是毋庸置疑的，但是我们代码中最为常用的模块并不多，其中所有的模块列表如下：

- **ribbon-core**:在生产中大量部署

- **ribbon-loadbalancer**:在生产中大规模部署。

- **ribbon-eureka**: 在生产中大规模部署

- **ribbon-evcache**:不使用

- **ribbon-guice**:不使用

- **ribbon-httpclient**:基本不使用

- **ribbon-transport**:不使用

主要的只要关注`ribbon-core`、`ribbon-loadbalancer`、`ribbon-eureka`这几个模块即可。而这几个模块都是和负载均衡有关。

> 1. Ribbon`客户端的启动原理是什么`
> 2. `Ribbon`如何自定义服务列表，如何自定义心跳策略
> 3. `Ribbon`如何修改器注册中心的解析策略

> 如何根据serviceId获取对应的loadbalance



BaseLoadBalancer中的ServerList和ConfigurationBasedServerList是什么关系

## 如何使用LoadBalance模块

### 1. 看一下官网的demo中是如何加载服务列表的

```java
public static void main(String[] args) {
  List<Server> serverList = new ArrayList<>();
  serverList.add(new Server("www.baidu.com", 80));
  serverList.add(new Server("www.google.com", 80));
  serverList.add(new Server("www.v2ex.com", 80));
  BaseLoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder().buildFixedServerListLoadBalancer(serverList);
  for (int i = 0; i < 6; i++) {
    LoadBalancerCommand<String> balancerCommand = LoadBalancerCommand.<String>builder()
      .withLoadBalancer(loadBalancer).build();
    ServerOperation<String> operation = server -> Observable.create(t->{
      t.onNext(server.getHost());
      t.onCompleted();
    });
    String single = balancerCommand.submit(operation).toBlocking().single();
    System.out.println(single);
  }
}
```

这里主要的逻辑很简单

- ***serverList这个变量保存了我们所有备选的server，就是我们上一节说的客户端负载均衡是在本地维护了一个服务列表***。
- ***BaseLoadBalancer是一个最基础的负载均衡器，仅提供基础的能力。我们把serverList交给负载均衡器***。
- ***LoadBalancerCommand我们暂且可以把它理解成对一个请求的封装***
- ***ServerOperation字面意思就是对server的操作。再挖深一点就是负载均衡器把选择好的Server给我们 我们拿着这个Server做什么。例如向这个Server发起Http请求？发TCP请求？这样是不是就通了？***



### 2. 最简易的方法如何定义了服务地址和负载均衡算法



- 配置文件中

```properties
client-test.ribbon.listOfServers=http://localhost:9000,http://localhost:32768
client-test.ribbon.NFLoadBalancerPingClassName=com.netflix.loadbalancer.PingUrl
client-test.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule
client-test.ribbon.NFLoadBalancerPingInterval=5
```

- 测试代码

```java
@Test
public void loadBalancerServer() {
  ServiceInstance instance = loadBalancerClient.choose("client-test");
  System.out.println("instance1:" + instance.getHost());
  ServiceInstance instance2 = loadBalancerClient.choose("client-test");
  System.out.println("instance2:" + instance2.getHost());
}
```



### 3.通过自定义的路由加载类、service-id等配置



- 重写ribbon.listOfServer配置

```java
/**
 * @Desc 自定义的ServerList，等价于重写ribbon.listOfServer配置
 * @Author wpstan
 * @Create 2019-12-31 22:20
 */
public class CustomServerList implements ServerList<Server>, IClientConfigAware {
    private IClientConfig clientConfig;

    @Override
    public List<Server> getInitialListOfServers() {
        return getUpdatedListOfServers();
    }

    @Override
    public List<Server> getUpdatedListOfServers() {
        List<Server> servers = new ArrayList<>();
        Server server1 = new Server("http://localhost:9003");
        Server server2 = new Server("http://localhost:9004");
        servers.add(server1);
        servers.add(server2);
        return servers;
    }

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {
        this.clientConfig = iClientConfig;
    }
}
```

- 自定义`ribbon`相关的配置类,可以自定义`IPing`、`IRule`、`ServerList`的实现

```java
/**
 * @Desc 自定义ribbon相关的配置类，可以在此类中构造IPing、IRule、ServerList等数据。
 *       可以从数据库配置中根据ribbon的service-id来构造不同的IPing、IRule、ServerList等实现类。
 *       当service-id没有构造的时候，会从此类中进行构造。
 * @Author wpstan
 * @Create 2019-12-31 22:21
 */
public class CustomRibbonClientConfiguration extends RibbonClientConfiguration {

    @Bean
    @ConditionalOnMissingBean
    @Override
    public IClientConfig ribbonClientConfig() {
        IClientConfig config = super.ribbonClientConfig();
        //可以增加一些配置，例如Ping后台主机的时间间隔5s
        config.set(CommonClientConfigKey.NFLoadBalancerPingInterval, 5);
        //可以在此设置Ping实现类（无需重写ribbonPing，base中会根据如下配置反射构造IPing），也可以使用方法直接构造，例如下面的ribbonPing()方法
        config.set(CommonClientConfigKey.NFLoadBalancerPingClassName,"com.wpstan.custom.ribbon.CustomPing");
        //设置MaxAutoRetries为2，确保某个服务挂的瞬间，ServerStats的Successive Connection Failure大于或等于
        //ServerStats中的connectionFailureThreshold的默认次数3。这样ServerStats的isCircuitBreakerTripped才被置位短路。
        //在Retry重试的时候，CustomRule中Choose下一个服务的时候，确保之前的服务已经短路，选用下一个服务，否则可能继续选择不可用服务导致报错。
        //还有一种方式就是调小ServerStats的connectionFailureThreshold。
        config.set(CommonClientConfigKey.MaxAutoRetries, 2);
        return config;
    }

    @Bean
    @ConditionalOnMissingBean
    @Override
    public ServerList<Server> ribbonServerList(IClientConfig config) {
        CustomServerList serverList = new CustomServerList();
        serverList.initWithNiwsConfig(config);
        return serverList;
    }

    @Bean
    @ConditionalOnMissingBean
    @Override
    public IRule ribbonRule(IClientConfig config) {
        CustomRule rule = new CustomRule();
        rule.initWithNiwsConfig(config);
        return rule;
    }

    @Bean
    @ConditionalOnMissingBean
    @Override
    public IPing ribbonPing(IClientConfig config) {
        return new CustomPing();
    }
}
```





## 组件介绍

![LoadBalancerClient](https://raw.githubusercontent.com/nezha/picdb/master/20210127202930.png)

主题还是分成两部分

### IRule

Ribbon中的用于自定义负载规则的

### IPing

用于自定义心跳策略

### ServerList

等价于重写ribbon.listOfServer配置

ConfigurationBasedServerList: 在配置文件中设置，如果属性时动态改变的，服务列表也会改变

DiscoveryEnabledNIWSServerList: 从eureka服务获取服务列表，服务集群必须由VipAddress标识

### ServerListUpdater

DynamicServerListLoadBalancer状态下，更新服务器列表

PollingServerListUpdater  默认，定时更新

EurekaNotificationServerListUpdater  收到通知时，更新服务器列表

### IClientConfig

定义各种配置信息，用来初始化ribbon客户端与负载均衡器

### ILoadBalancer

定义软负载均衡的操作接口，动态更新服务列表并且根据指定策略算法，从服务列表中选择一个服务

BaseLoadBalancer

DynamicServerListLoadBalancer

ZoneAwareLoadBalancer

### LoadBalancerClient

发起负载均衡的入口,其实现类是：RibbonLoadBalancerClient



### RibbonClientConfiguration

Ribbon的初始化配置类

## 几个过程

### 1.初始化读取配置过程

### 2.负载均衡选择服务列表

### 3.服务列表更新





## 类之间的关系



```
RibbonLoadBalancerClient
ILoadBalancer

```





## 如何通过`Ribbon`加载注册中心的`Servers`

> 1. 可以肯定的是第一次的通过RibbonLoadBalancerClient的choose调用时才会初始化RibbonClientConfiguration
> 2. 





## 参考文献



1.  [【你好Ribbon】系列](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&__biz=MzkzOTE3NDcyNg==&scene=24&album_id=1585076049778966529&count=3#wechat_redirect)
2.  [Ribbon 负载均衡器 LoadBalancer 源码解析](https://my.oschina.net/mengyuankan/blog/3104184)
3.  