---
title: Hystrix Finchley
date: 2018-12-18 17:05:21
tags:
---
# Spring Cloud Hystrix 配置

Spring Cloud Finchley + Spring boot 2.0.6

本文是在已经实现了openfeign整合hystrix，并且实现了利用hystrixDashboard对hystrix的图形化监控。记录下我在实现过程中碰到的坑。

## 打开hystrix支持

官网上说的时候引入openfeign后如果hystrix在依赖中，则会自动将feignclient包装成一个hystrixCommond，使用命令模式可以降低调用者和执行者之间的耦合度，并且利于回退等事务式的管理。实际上在配置文件中仍然需要配置
```java
feign.hystrix.enabled=true
```
才能使得回退生效

## 高大上的hystrix

hystrix借鉴docker的舱壁隔离模式，为每一个服务开启独立的线程池，避免服务调用失败占用过多系统资源无法释放，影响了其他的服务。这种设计模式导致为每个服务分配的系统资源可能会不合适而导致系统负载不均衡。不过设计者认为这点代价相对于获得便利是值得的。

hystrix官网上提到了一个隔离策略（execution.isolation.strategy），这个隔离策略只有两种，一种按照线程隔离，一种按照信号量隔离。按照线程隔离更安全，按照信号量能提供更高的并发。

![hystrix隔离策略](https://duaw26jehqd4r.cloudfront.net/items/3e0W2E1k3b2b2d2u0p1L/soa-5-isolation-focused-640.png)

## 熔断器原理

断路器的实现原理是在一段时间内超过指定请求数量有超过指定比例请求失败，则会开启断路器，使得断路器进入打开状态，后续的请求将不再发起实际请求，而是直接返回fallback。
```java
circuitBreaker.requestVolumeThreshold=20
circuitBreaker.errorThresholdPercentage=50
circuitBreaker.sleepWindowInMilliseconds=5000
metrics.rollingStats.timeInMilliseconds=10000
```
以上配置的意思是：

当hystrix遇到调用资源失败，会开启一个长度为10s的时间窗口，在这个时间窗口内，检测所有请求，如果失败数量大于20，并且比例大于50%就会开启断路器。断路器持续5s，5s后断路器进入半开状态，允许下一个请求发起调用，如果成功断路器关闭，否则继续打开。

## hystrix 监控

直接运行项目，访问/hystrix.stream会得到一个404。这是因为首先，在actuator 2以后所有的端点都加了默认前缀/actuator，当然这个前缀是可以修改的。其次，因为端点会暴露很多敏感信息，所以web方式暴露的端点默认只有两个：health，info.开启其他端点需要通过配置：
```java
management.endpoints.web.exposure.include=*
```
在yml文件中*有特殊含义，所以需要加上引号。如果只想开放指定端点，也可以在后面配置一个字符串数组，只包含自己想要开放的端点。

同样的道理，在spring cloud config 配置的时候想要通过/refresh刷新配置，也会得到404.

然后在主类上加上注解
```java
@EnableCircuitBreaker
@EnableHystrix
public class ConsumerApplication {
}
```
虽然已经打开了feign对hystrix的支持，但是没有这两个注解，访问/actuator/hystrix.stream还是会报错的。加上之后再访问，会一直ping。这样就对了。随后就是按照网上的一半教程搭建一个hystrixDashboard项目就能实现对hystrix的监控。在监控页面也可以通过circuit来判断断路器是否打开，根据前面说的断路器原理，回退了但是不一定会打开断路器。