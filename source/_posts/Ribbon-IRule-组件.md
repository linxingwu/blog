---
title: Ribbon IRule 组件
date: 2018-12-28 09:30:30
tags:
 - Spring Cloud
 - Ribbon
 - IRule
 - Finchley
---
前文说了ILoadbalancer的两个重要实现，这次说一下IRule的主要实现。

按照前文提到过的Ribbon的一贯个性，首先是一个最高级的接口IRule，定义了一些基本的功能。

然后一个抽象实现类AbstractLoadBalancerRule,抽象类增加了一个负载均衡器（服务实例列表）用来实现具体的选择逻辑。

# RoundRobinRule

然后一个base的实现类，这一层是大多数都有的。但并不是一定有，如果非要套上去的话，我认为应该是RoundRobinRule。它是默认的实现，通过轮询的方式来选择实例。


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

它的代码很简单，之前也简单提到过，也没什么可深究的，就是遍历。

## WeightedResponseTimeRule

继承自上一个类，实现了按照响应时间计算权重，然后按照权重选择服务实例的功能。作者对此有一段说明还是很有用的：

>Rule that use the average/percentile response times to assign dynamic "weights" per Server which is then used in the "Weighted Round Robin" fashion.
>The basic idea for weighted round robin has been obtained from JCS The implementation for choosing the endpoint from the list of endpoints is as follows:Let's assume 4 endpoints:A(wt=10), B(wt=30), C(wt=40), D(wt=20).
>Using the Random API, generate a random number between 1 and10+30+40+20. Let's assume that the above list is randomized. Based on the weights, we have intervals as follows:

>1-----10 (A's weight)

>11----40 (A's weight + B's weight) 

>41----80 (A's weight + B's weight + C's weight) 

>81----100(A's weight + B's weight + C's weight + C's weight)

>Here's the psuedo code for deciding where to send the request:
```java
if (random_number between 1 & 10) {
    send request to A;
} else if (random_number between 11 & 40) {
    send request to B;
} else if (random_number between 41 & 80) {
    send request to C;
} else if (random_number between 81 & 100) {
    send request to D;
}
```
>When there is not enough statistics gathered for the servers, this rule will fall back to use RoundRobinRule.

计算思路就是前一个权重作为本server的下限，加上本区间的权重得出server的上限，然后产生一个随机数，落在谁的区间就选择谁。第一个的下限是0.那么权重是怎么计算出来的呢？

WeightedResponseTimeRule中init时启动了一个定时任务每隔30s运行一次。启动配置部分略过了，直接看定时任务做了什么操作

```
        public void maintainWeights() {
            ILoadBalancer lb = getLoadBalancer();
            if (lb == null) {
                return;
            }
            
            if (!serverWeightAssignmentInProgress.compareAndSet(false,  true))  {
                return; 
            }
            
            try {
                logger.info("Weight adjusting job started");
                AbstractLoadBalancer nlb = (AbstractLoadBalancer) lb;
                LoadBalancerStats stats = nlb.getLoadBalancerStats();
                if (stats == null) {
                    // no statistics, nothing to do
                    return;
                }
                double totalResponseTime = 0;
                // find maximal 95% response time
                for (Server server : nlb.getAllServers()) {
                    // this will automatically load the stats if not in cache
                    ServerStats ss = stats.getSingleServerStat(server);
                    totalResponseTime += ss.getResponseTimeAvg();
                }
                // weight for each server is (sum of responseTime of all servers - responseTime)
                // so that the longer the response time, the less the weight and the less likely to be chosen
                Double weightSoFar = 0.0;
                
                // create new list and hot swap the reference
                List<Double> finalWeights = new ArrayList<Double>();
                for (Server server : nlb.getAllServers()) {
                    ServerStats ss = stats.getSingleServerStat(server);
                    double weight = totalResponseTime - ss.getResponseTimeAvg();
                    weightSoFar += weight;
                    finalWeights.add(weightSoFar);   
                }
                setWeights(finalWeights);
            } catch (Exception e) {
                logger.error("Error calculating server weights", e);
            } finally {
                serverWeightAssignmentInProgress.set(false);
            }

        }
```
首先定义了一个double totalResponseTime 用来统计所有服务实例的累积平均响应时间。然后用总的时间-本实例平均响应时间，计算出本实例的权重，也就是说响应时间越久权重越轻。再用一个weightSoFar来计算每一个server的上限，同时也是下一个server的下限。存在一个有序的list中，这样和serverList就能按照index一一对应。

计算好权重区间之后看看怎么使用的
```
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            // get hold of the current reference in case it is changed from the other thread
            List<Double> currentWeights = accumulatedWeights;
            if (Thread.interrupted()) {
                return null;
            }
            List<Server> allList = lb.getAllServers();

            int serverCount = allList.size();

            if (serverCount == 0) {
                return null;
            }

            int serverIndex = 0;

            // last one in the list is the sum of all weights
            double maxTotalWeight = currentWeights.size() == 0 ? 0 : currentWeights.get(currentWeights.size() - 1); 
            // No server has been hit yet and total weight is not initialized
            // fallback to use round robin
            if (maxTotalWeight < 0.001d || serverCount != currentWeights.size()) {
                server =  super.choose(getLoadBalancer(), key);
                if(server == null) {
                    return server;
                }
            } else {
                // generate a random weight between 0 (inclusive) to maxTotalWeight (exclusive)
                double randomWeight = random.nextDouble() * maxTotalWeight;
                // pick the server index based on the randomIndex
                int n = 0;
                for (Double d : currentWeights) {
                    if (d >= randomWeight) {
                        serverIndex = n;
                        break;
                    } else {
                        n++;
                    }
                }

                server = allList.get(serverIndex);
            }

            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Next.
            server = null;
        }
        return server;
    }
```

accumulatedWeights 里存的就是定时任务计算的结果。选择的原理就是在[0,最大响应时间)里产生一个随机数，然后遍历accumulatedWeights，如果同index的server的上限>=这个随机数，这个server就是被选中的幸运儿。

当然也有例外情况，如果发现服务实例数和计算的权重区间数不吻合说明计算出错了，或者累计的响应时间小于0.1说明定时任务还没开始，那就直接调用父类的轮询选择策略。

# RandomRule

这是一个光杆的base，说它是base因为它位于抽象类下，但是又没有其他人来继承它。毕竟看名字也知道，随机选择一个太没有技术含量，low的很。


# RetryRule

这个类并没有继承RoundRobinRule，但是它玩了一个代理，代理了RoundRobinRule。然后在实现choose的时候，添加了一个时间限制，默认是500ms，在500ms内轮询选的server如果不好使，还能包换，再选一个，一直循环到找一个好使的。

# ClientConfigEnabledRoundRobinRule

这个类同样是一个RoundRobinRule的代理类，它甚至没有任何创新，完全使用的RoundRobinRule的规则。那么它有什么存在的价值呢？大佬认为假如我们自己实现了一个Rule，就WeightedResponseTimeRule好了，它现在是你实现的了，你考虑到你的权重区间没计算好或者计算错了的时候要调用父类的备选策略，你就可以继承ClientConfigEnabledRoundRobinRule，抛给它的默认实现。事实上后续的几个高级Rule也是这么操作的。

但是我就不明白，那我直接继承RoundRobinRule有什么问题呢？何必继承ClientConfigEnabledRoundRobinRule？

## BestAvailableRule

前者的一个子类，实现的功能就是根据并发请求数量选择最空闲的一个server。代码很简单，没有分析的必要了。

```java
    public Server choose(Object key) {
        if (loadBalancerStats == null) {
            return super.choose(key);
        }
        List<Server> serverList = getLoadBalancer().getAllServers();
        int minimalConcurrentConnections = Integer.MAX_VALUE;
        long currentTime = System.currentTimeMillis();
        Server chosen = null;
        for (Server server: serverList) {
            ServerStats serverStats = loadBalancerStats.getSingleServerStat(server);
            if (!serverStats.isCircuitBreakerTripped(currentTime)) {
                int concurrentConnections = serverStats.getActiveRequestsCount(currentTime);
                if (concurrentConnections < minimalConcurrentConnections) {
                    minimalConcurrentConnections = concurrentConnections;
                    chosen = server;
                }
            }
        }
        if (chosen == null) {
            return super.choose(key);
        } else {
            return chosen;
        }
    }
```
## PredicateBasedRule

这在本文中，又是一个递归，这个类是一个抽象类，实现了自定义条件选择server。
```java
public abstract class PredicateBasedRule extends ClientConfigEnabledRoundRobinRule {
   
    /**
     * Method that provides an instance of {@link AbstractServerPredicate} to be used by this class.
     * 
     */
    public abstract AbstractServerPredicate getPredicate();
        
    /**
     * Get a server by calling {@link AbstractServerPredicate#chooseRandomlyAfterFiltering(java.util.List, Object)}.
     * The performance for this method is O(n) where n is number of servers to be filtered.
     */
    @Override
    public Server choose(Object key) {
        ILoadBalancer lb = getLoadBalancer();
        Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
        if (server.isPresent()) {
            return server.get();
        } else {
            return null;
        }       
    }
}
```
没错，这里没有明显调用过滤条件，真正调用的在AbstractServerPredicate里，我们根据chooseRoundRobinAfterFiltering也能知道，这个方法先过滤，然后过滤结果中轮询一个。以下是过滤的实现
```java
    public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
        if (loadBalancerKey == null) {
            return ImmutableList.copyOf(Iterables.filter(servers, this.getServerOnlyPredicate()));            
        } else {
            List<Server> results = Lists.newArrayList();
            for (Server server: servers) {
                if (this.apply(new PredicateKey(loadBalancerKey, server))) {
                    results.add(server);
                }
            }
            return results;            
        }
    }
```
这里Predicate的高级之处前文也提到过，显而易见，典型的模板模式，先定好，咱们都是先过滤，然后再选择，具体怎么过滤，孩子们自己决定，这样以后扩展方便多了。

### AvailabilityFilteringRule

继承了PredicateBasedRule，看看它是怎么设置Predicate的：
```java
    public AvailabilityFilteringRule() {
    	super();
    	predicate = CompositePredicate.withPredicate(new AvailabilityPredicate(this, null))
                .addFallbackPredicate(AbstractServerPredicate.alwaysTrue())
                .build();
    }
```
这段代码应该也能看明白，就是创建了一个复合的Predicate，先AvailabilityPredicate，然后再加一道没什么意义的过滤，因为后者实际上总是返回true。

但是注意到，build返回的是一个CompositePredicate，它和Predicate还是有点不同的，

```java
    @Override
    public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
        List<Server> result = super.getEligibleServers(servers, loadBalancerKey);
        Iterator<AbstractServerPredicate> i = fallbacks.iterator();
        while (!(result.size() >= minimalFilteredServers && result.size() > (int) (servers.size() * minimalFilteredPercentage))
                && i.hasNext()) {
            AbstractServerPredicate predicate = i.next();
            result = predicate.getEligibleServers(servers, loadBalancerKey);
        }
        return result;
    }
```
可以看到CompositePredicate有一个list的Predicate备选过滤条件，如果上次过滤后剩下的数量和百分比都还理想，它会把所有的备选都拉出来溜溜，直到某一个predicate过滤后数量，百分比不理想了





我们的重点现在转移到了AvailabilityPredicate，看看它是怎么实现的
```java
    @Override
    public boolean apply(@Nullable PredicateKey input) {
        LoadBalancerStats stats = getLBStats();
        if (stats == null) {
            return true;
        }
        return !shouldSkipServer(stats.getSingleServerStat(input.getServer()));
    }
    
    
    private boolean shouldSkipServer(ServerStats stats) {        
        if ((CIRCUIT_BREAKER_FILTERING.get() && stats.isCircuitBreakerTripped()) 
                || stats.getActiveRequestsCount() >= activeConnectionsLimit.get()) {
            return true;
        }
        return false;
    }
```

根据断路器和并发量，选择没有触发断路器，并发量没有超标的server，这个标的默认值是Integer.MAX_VALUE，当然是可以自定义的。

AvailabilityFilteringRule同时也对choose做出了一些小的贡献
```java
    @Override
    public Server choose(Object key) {
        int count = 0;
        Server server = roundRobinRule.choose(key);
        while (count++ <= 10) {
            if (predicate.apply(new PredicateKey(server))) {
                return server;
            }
            server = roundRobinRule.choose(key);
        }
        return super.choose(key);
    }
```
父类的choose里调用了getEligibleServers，总是在遍历，难免带着老一辈人的落后思想。这里先尝试10次自己的先进思想：轮询，过滤，合适就返回。尝试失败再走父辈留下的后路。

### ZoneAvoidanceRule

也是一个复合过滤条件，和前文的AvailabilityFilteringRule大同小异了
```java
    private CompositePredicate compositePredicate;
    
    public ZoneAvoidanceRule() {
        super();
        ZoneAvoidancePredicate zonePredicate = new ZoneAvoidancePredicate(this);
        AvailabilityPredicate availabilityPredicate = new AvailabilityPredicate(this);
        compositePredicate = createCompositePredicate(zonePredicate, availabilityPredicate);
    }
    
```

