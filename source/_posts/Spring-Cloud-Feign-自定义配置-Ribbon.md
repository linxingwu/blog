---
title: Spring Cloud Feign 自定义配置 Ribbon
date: 2018-12-14 17:06:17
tags:
---
# Spring Cloud 自定义配置Feign

Spring Cloud Finchley + Spring boot 2.0.6

## 自定义ribbon配置

网上有很多相关介绍，官网的[相关文档](https://cloud.spring.io/spring-cloud-static/Finchley.SR2/multi/multi_spring-cloud.html)也是正确的。本次选用的通过自定义LoadBalancer实现类的配置方式作为实验，其他的Rule,Ping都是同理。
```java

import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyRibbonConfig  {

    @Bean
    public ILoadBalancer ribbonLoadBalancer() {
        return new DynamicServerListLoadBalancer<>();
    }

}
```
为了只影响目标服务，不影响其他服务，我们需要**把配置类移出ComponentScan的范围**

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
@ComponentScan(excludeFilters = {@ComponentScan.Filter (type=FilterType.ASSIGNABLE_TYPE, classes = MyRibbonConfig.class)})
public class ConsumerApplication {}
```
没错就是这么简单粗暴，在主类上增加一个@ComponentScan排除掉自定义的配置类。其中ASSIGNABLE_TYPE表示按照类来过滤，class指定要过滤的类。当然在SpringBootApplication中也含有这个注解
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}
```
所以也可以通过继承TypeExcludeFilter重写match方法来过滤掉配置类，稍微麻烦一些。

随后我们通过@FeignClient指定我们的配置文件和针对的具体服务：
```java
@Configuration
@RibbonClient(name="hello-service",configuration = MyRibbonConfig.class)
public class TestRibbonConfiguration {
}
```
## 代码出错

启动程序之后发现会报错，找不到server
```java
Caused by: java.lang.RuntimeException: com.netflix.client.ClientException: Load balancer does not have available server for client: hello-service
	at org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient.execute(LoadBalancerFeignClient.java:71) ~[spring-cloud-openfeign-core-2.0.1.RELEASE.jar:2.0.1.RELEASE]
	at org.springframework.cloud.sleuth.instrument.web.client.feign.TraceLoadBalancerFeignClient.execute(TraceLoadBalancerFeignClient.java:67) ~[spring-cloud-sleuth-core-2.0.1.RELEASE.jar:2.0.1.RELEASE]
```

## 跟踪源码

没有办法，只好根据报错信息，一路跳啊跳，最终调到了IRule的实现类中，调用了choose方法
```java
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
            //这里会发现lb里的servers是空的，下面就会返回一个错误
            List<Server> reachableServers = lb.getReachableServers();
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();

            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }

            int nextServerIndex = incrementAndGetModulo(serverCount);
            server = allServers.get(nextServerIndex);

            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }

            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            // Next.
            server = null;
        }

        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }
```

那么这个servers是从哪里来的呢？或者说是什么时候注入进去的呢？我在[网上的源码分析](http://blog.didispace.com/springcloud-sourcecode-ribbon/)里找了半天反正是没有找明白。不过既然我已经使用了springcloud自带的loadbalancer还是会出错，那么真相只有一个，我配置出问题了。

这篇文章不是基于Finchley版本的，和我的代码有很大出入，我是没看完，不过我从中得到一个很重要的信息,默认的LoadBalancer,RibbonClientConfig是如何初始化的
```java
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
```
## 注入server

这里有直接注入的serverlist，既然如此，我也修改一下我的配置类，不调用无参构造方法，直接调用有参构造，为了更彻底的测试我继承spring的loadbalancer
```java
@Configuration
public class MyRibbonConfig  {

    @Bean
    public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
                                            ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
                                            IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
        return new MyLoadBalancer(config, rule, ping, serverList,
                serverListFilter, serverListUpdater);
    }

}

//必须使用有参构造方法传入serverlist，否则调用找不到server报错
class MyLoadBalancer extends DynamicServerListLoadBalancer{
    MyLoadBalancer(IClientConfig config, IRule rule, IPing ping, ServerList<Server> serverList, ServerListFilter<Server> serverListFilter, ServerListUpdater serverListUpdater) {
        super(config,rule,ping,serverList,serverListFilter,serverListUpdater);
    }

}
```

随后测试，效果拔群。