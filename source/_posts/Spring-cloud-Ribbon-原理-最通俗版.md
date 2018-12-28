---
title: Spring cloud Ribbon 原理最通俗版
date: 2018-12-26 14:09:01
tags:
---
# Ribbon 配置

因为主要记录我对Ribbon原理的理解(来源于翟永超的《Spring Cloud 微服务实战》)，看了很长时间，最后还是云里雾里的，原作者在梳理整个流程的时候穿插了很多各个组件的详细代码分析，但是又不能照顾到所有人，所以必然看了后面就忘了前面。这篇文章考验了我自己的理解，希望能帮到其他人

网上有很多关于Ribbon的配置教程，所以本文不再赘述Ribbon的详细配置。而是直接从这里开始：
```java

    @Bean
    @LoadBalanced
    RestTemplate restTemplate(){
        return new RestTemplate();
    }
```

RestTemplate是springframework.web里自带用于rest通信的类，加上@LoadBalanced就能出现神奇的效果。

# 原理

Ribbon的配置类主要靠LoadBalancerAutoConfiguration实现。主要是把restTemplates加一个拦截器，然后拦截调用过程，做了一顿操作，实现的负载均衡。
```java
    @Bean
    @ConditionalOnMissingBean
    public RestTemplateCustomizer restTemplateCustomizer(
            final LoadBalancerInterceptor loadBalancerInterceptor) {
        return restTemplate -> {
            List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                    restTemplate.getInterceptors());
            list.add(loadBalancerInterceptor);
            restTemplate.setInterceptors(list);
        };
    }

```

拦截器的实现

```java

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
```
loadBalancer是一个是一个客户端负载均衡的抽象借口LoadBalancerClient的实现，@Loadbalancer的注释说的就是将一个restTemplate变成一个LoadBalancerClient的实现

```java

public interface LoadBalancerClient extends ServiceInstanceChooser {

	/**
	 * 从负载均衡器中选择一个特定服务的实例来执行请求
	 * @param serviceId 负载均衡器要寻找的服务id
	 * @param request allows implementations to execute pre and post actions such as
	 * incrementing metrics
	 * @return the result of the LoadBalancerRequest callback on the selected
	 * ServiceInstance
	 */
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

	/**
	 * execute request using a ServiceInstance from the LoadBalancer for the specified
	 * service
	 * @param serviceId the service id to look up the LoadBalancer
	 * @param serviceInstance the service to execute the request to
	 * @param request allows implementations to execute pre and post actions such as
	 * incrementing metrics
	 * @return the result of the LoadBalancerRequest callback on the selected
	 * ServiceInstance
	 */
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

	/**
	 * Create a proper URI with a real host and port for systems to utilize.
	 * Some systems use a URI with the logical serivce name as the host,
	 * such as http://myservice/path/to/service.  This will replace the
	 * service name with the host:port from the ServiceInstance.
	 * @param instance
	 * @param original a URI with the host as a logical service name
	 * @return a reconstructed URI
	 */
	URI reconstructURI(ServiceInstance instance, URI original);
}
```

这个接口已经和书上有很大的变化了，去掉了choose方法，我猜想这个主要是设计用来选择服务的，但是现在改用了IRule的choose，所以这个方法去掉了。

基于以上的理解，我在interceptor上打一个断点，就能更好的了解下一步如何执行了。当然用f8一步一步来调试也是不合适的，所以我在配置的时候故意将serverList设为空的，这样肯定会报找不到serverInstance的异常，然后根据错误信息找到出错的一行，打上断点，运行到这里的时候通过idea查看调用的堆栈信息来补充上另外几个断点以防万一。


跟随断点可以看到拦截器拦截的请求后，调用了LoadBalancer的execute方法，这里的LoadBalancer毫无疑问的就是RibbonLoadBalancer了，它具体实现是这样的
```java
	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
		Server server = getServer(loadBalancer);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
				serviceId), serverIntrospector(serviceId).getMetadata(server));

		return execute(serviceId, ribbonServer, request);
	}
	
    protected Server getServer(ILoadBalancer loadBalancer) {
        if (loadBalancer == null) {
            return null;
        }
        return loadBalancer.chooseServer("default"); // TODO: better handling of key
    }
    
    @Override
    public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
        Server server = null;
        if(serviceInstance instanceof RibbonServer) {
            server = ((RibbonServer)serviceInstance).getServer();
        }
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }

        RibbonLoadBalancerContext context = this.clientFactory
                .getLoadBalancerContext(serviceId);
        RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

        try {
            T returnVal = request.apply(serviceInstance);
            statsRecorder.recordStats(returnVal);
            return returnVal;
        }
        // catch IOException and rethrow so RestTemplate behaves correctly
        catch (IOException ex) {
            statsRecorder.recordStats(ex);
            throw ex;
        }
        catch (Exception ex) {
            statsRecorder.recordStats(ex);
            ReflectionUtils.rethrowRuntimeException(ex);
        }
        return null;
    }
```
实现的思路就是通过ILoadBalancer的chooseServer方法选择一个具体的server，然后调用execute发起具体的请求，返回请求结果。发起的请求过程中包含了将serverId转换为具体的Url的过程，实际最终调用了LoadBalancer的reconstructURI方法。
```java

	@Override
	public URI reconstructURI(ServiceInstance instance, URI original) {
		Assert.notNull(instance, "instance can not be null");
		String serviceId = instance.getServiceId();
		RibbonLoadBalancerContext context = this.clientFactory
				.getLoadBalancerContext(serviceId);

		URI uri;
		Server server;
		if (instance instanceof RibbonServer) {
			RibbonServer ribbonServer = (RibbonServer) instance;
			server = ribbonServer.getServer();
			uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
		} else {
			server = new Server(instance.getScheme(), instance.getHost(), instance.getPort());
			IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
			ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
			uri = updateToSecureConnectionIfNeeded(original, clientConfig,
					serverIntrospector, server);
		}
		return context.reconstructURIWithServer(server, uri);
	}
```
所以重点就在于负载均衡器是如何选择一个具体Server的。
```java
    public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
                return rule.choose(key);
            } catch (Exception e) {
                logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
                return null;
            }
        }
    }
```
这是Loadbalancer的一个具体实现，可以看到它并没有做什么建设的工作，而是直接把任务委托给了IRule来选择，所以原来这个接口中的chooseServer后来被删除了。这里的这个IRule是默认的实现，RoundRobinRule。我们看看它的choose方法是如何实现的
```java
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
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
每一个IRule中都维护了一个负载均衡器，所谓负载均衡器就是可以简单理解为一个server的list，它就维护了所有的server,顺便提供一些选择的方法。这样一想就简单多了，不然容易绕晕。原作者的解释是

> Interface that defines the operations for a software loadbalancer. A typical loadbalancer minimally need a set of servers to loadbalance for, a method to mark a particular server to be out of rotation and a call that will choose a server from the existing list of server.
 
RoundRobinRule中维护了一个AtomicInteger类型的index，用来表示当前轮询到的server位置，同时在方法内部定义了一个int的轮询次数，如果超过10次就直接返回null了。每次轮询的过程实际上就是把index+1，然后尝试返回这个server，尝试的过程就是对server的检验，是否还活着是否还能提供服务。



以上就是最简单的一版关于Ribbon原理的说明，简单来说就是通过@LoadBalanced把restTemplate添加一个拦截器，在拦截器中调用负载均衡器的execute方法来发起具体的请求，负载均衡器又通过IRule来实现serverid到具体的ip地址的转换，转换的过程也是一个负载均衡的过程。下一篇我再详细讲解各个组件具体的实现原理。
 
 