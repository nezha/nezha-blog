# Hystrix学习笔记一

> Hystrix作为服务熔断降级的组件，在生产中已经有了大面积的使用和推广，本文将针对Hystrix使用过程中的重点和难点进行学习。




## 熔断器的常用方法


## 如何控制熔断粒度
> 目前识别的可控粒度有：
> `GroupKey`
> `CommandKey`
> `ThreadPoolKey`



## 熔断器的关键参数
> https://github.com/Netflix/Hystrix/wiki/Configuration






## 参考文献



1. [HystrixCommand的threadPoolKey默认值及线程池初始化](https://www.cnblogs.com/trust-freedom/p/9956427.html)
2. [Hystrix常用功能介绍](https://segmentfault.com/a/1190000012549823)
3. [Hystrix官方配置解释](https://github.com/Netflix/Hystrix/wiki/Configuration)
4. [Hystrix熔断器技术解析-HystrixCircuitBreaker](https://www.jianshu.com/p/14958039fd15)
5. [Hystrix中Metrics关键指标](https://blog.csdn.net/wk52525/article/details/90550171)