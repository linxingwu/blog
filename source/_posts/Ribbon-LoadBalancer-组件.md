---
title: Ribbon LoadBalancer 组件 
date: 2018-12-27 11:17:17
tags: 
 - Spring Cloud 
 - Ribbon
---
# ILoadBalancer

这是最高级的一个抽象类.它有一个抽象的实现AbstractLoadBalancer.

```java

public interface ILoadBalancer {

	/**
	 * Initial list of servers.
	 * This API also serves to add additional ones at a later time
	 * The same logical server (host:port) could essentially be added multiple times
	 * (helpful in cases where you want to give more "weightage" perhaps ..)
	 * 
	 * @param newServers new servers to add
	 */
	public void addServers(List<Server> newServers);
	
	/**
	 * Choose a server from load balancer.
	 * 
	 * @param key An object that the load balancer may use to determine which server to return. null if 
	 *         the load balancer does not use this parameter.
	 * @return server chosen
	 */
	public Server chooseServer(Object key);
	
	/**
	 * To be called by the clients of the load balancer to notify that a Server is down
	 * else, the LB will think its still Alive until the next Ping cycle - potentially
	 * (assuming that the LB Impl does a ping)
	 * 
	 * @param server Server to mark as down
	 */
	public void markServerDown(Server server);
	
	/**
	 * @deprecated 2016-01-20 This method is deprecated in favor of the
	 * cleaner {@link #getReachableServers} (equivalent to availableOnly=true)
	 * and {@link #getAllServers} API (equivalent to availableOnly=false).
	 *
	 * Get the current list of servers.
	 *
	 * @param availableOnly if true, only live and available servers should be returned
	 */
	@Deprecated
	public List<Server> getServerList(boolean availableOnly);

	/**
	 * @return Only the servers that are up and reachable.
     */
    public List<Server> getReachableServers();

    /**
     * @return All known servers, both reachable and unreachable.
     */
	public List<Server> getAllServers();
}
```

## AbstractLoadBalancer

```java
public abstract class AbstractLoadBalancer implements ILoadBalancer {
    
    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP        
    }
        
    /**
     * delegate to {@link #chooseServer(Object)} with parameter null.
     */
    public Server chooseServer() {
    	return chooseServer(null);
    }

    
    /**
     * List of servers that this Loadbalancer knows about
     * 
     * @param serverGroup Servers grouped by status, e.g., {@link ServerGroup#STATUS_UP}
     */
    public abstract List<Server> getServerList(ServerGroup serverGroup);
    
    /**
     * Obtain LoadBalancer related Statistics
     */
    public abstract LoadBalancerStats getLoadBalancerStats();    
}
```
这个抽象类中设置了一个enum用来表示不同状态是serverList，然后就是增加了一个无参的chooseServer。

### BaseLoadBalancer

这是第一个具体的实现类也基本是所有其他负载均衡器的父类。看一下它都是怎么构成了，做了什么
```java
    private final static IRule DEFAULT_RULE = new RoundRobinRule();
    private final static SerialPingStrategy DEFAULT_PING_STRATEGY = new SerialPingStrategy();
    private static final String DEFAULT_NAME = "default";
    private static final String PREFIX = "LoadBalancer_";

    protected IRule rule = DEFAULT_RULE;

    protected IPingStrategy pingStrategy = DEFAULT_PING_STRATEGY;

    protected IPing ping = null;

    @Monitor(name = PREFIX + "AllServerList", type = DataSourceType.INFORMATIONAL)
    protected volatile List<Server> allServerList = Collections
            .synchronizedList(new ArrayList<Server>());
    @Monitor(name = PREFIX + "UpServerList", type = DataSourceType.INFORMATIONAL)
    protected volatile List<Server> upServerList = Collections
            .synchronizedList(new ArrayList<Server>());
    protected ReadWriteLock allServerLock = new ReentrantReadWriteLock();
    protected ReadWriteLock upServerLock = new ReentrantReadWriteLock();

    protected String name = DEFAULT_NAME;

    protected Timer lbTimer = null;
    protected int pingIntervalSeconds = 10;
    protected int maxTotalPingTimeSeconds = 5;
    protected Comparator<Server> serverComparator = new ServerComparator();

    protected AtomicBoolean pingInProgress = new AtomicBoolean(false);

    protected LoadBalancerStats lbStats;
```
1. IRule

    默认的IRule就是RoundRobinRule，

2. SerialPingStrategy 
    ```java
        private static class SerialPingStrategy implements IPingStrategy {
    
            @Override
            public boolean[] pingServers(IPing ping, Server[] servers) {
                int numCandidates = servers.length;
                boolean[] results = new boolean[numCandidates];
    
                logger.debug("LoadBalancer:  PingTask executing [{}] servers configured", numCandidates);
    
                for (int i = 0; i < numCandidates; i++) {
                    results[i] = false; /* Default answer is DEAD. */
                    try {
                        // NOTE: IFF we were doing a real ping
                        // assuming we had a large set of servers (say 15)
                        // the logic below will run them serially
                        // hence taking 15 times the amount of time it takes
                        // to ping each server
                        // A better method would be to put this in an executor
                        // pool
                        // But, at the time of this writing, we dont REALLY
                        // use a Real Ping (its mostly in memory eureka call)
                        // hence we can afford to simplify this design and run
                        // this
                        // serially
                        if (ping != null) {
                            results[i] = ping.isAlive(servers[i]);
                        }
                    } catch (Exception e) {
                        logger.error("Exception while pinging Server: '{}'", servers[i], e);
                    }
                }
                return results;
            }
        }
    ```
    采用了一种遍历的方式去ping 所有的server。作者的注释也指出，这种效率并不高。应当放在线程池里执行。代码里这么写，是因为IPing的实现，并没有真的去ping。

3. IPing
    ```java
    public class DummyPing extends AbstractLoadBalancerPing {
    
        public DummyPing() {
        }
    
        public boolean isAlive(Server server) {
            return true;
        }
    
        @Override
        public void initWithNiwsConfig(IClientConfig clientConfig) {
        }
    }
    ```
    这个已经和书上说的不一样了，官网说的DummyPing，书上说的NoOpPing，不过区别不大，都是永远返回true。

4. 维护了两个server的list，一个是正常服务，一个是所有服务。

5. LoadBalancerStats

    这个类中存储了一些server的统计信息，断路器状态等等，用于辅助做出决策。例如在WeightedResponseTimeRule中计算权重就是根据这个类提供的信息。

6. 定时任务
    在默认的无参构造方法中启动了一个定时任务，每隔10s运行一次。此类的无参构造只是用于反射，日常初始化还需要调用init等其他方法，所以如果要使用这个类，不应该直接调用无参构造方法。
    ```java
    public void runPinger() throws Exception {
        if (!pingInProgress.compareAndSet(false, true)) { 
            return; // Ping in progress - nothing to do
        }
        
        // we are "in" - we get to Ping

        Server[] allServers = null;
        boolean[] results = null;

        Lock allLock = null;
        Lock upLock = null;

        try {
            /*
             * The readLock should be free unless an addServer operation is
             * going on...
             */
            allLock = allServerLock.readLock();
            allLock.lock();
            allServers = allServerList.toArray(new Server[allServerList.size()]);
            allLock.unlock();

            int numCandidates = allServers.length;
            results = pingerStrategy.pingServers(ping, allServers);

            final List<Server> newUpList = new ArrayList<Server>();
            final List<Server> changedServers = new ArrayList<Server>();

            for (int i = 0; i < numCandidates; i++) {
                boolean isAlive = results[i];
                Server svr = allServers[i];
                boolean oldIsAlive = svr.isAlive();

                svr.setAlive(isAlive);

                if (oldIsAlive != isAlive) {
                    changedServers.add(svr);
                    logger.debug("LoadBalancer [{}]:  Server [{}] status changed to {}", 
                        name, svr.getId(), (isAlive ? "ALIVE" : "DEAD"));
                }

                if (isAlive) {
                    newUpList.add(svr);
                }
            }
            upLock = upServerLock.writeLock();
            upLock.lock();
            upServerList = newUpList;
            upLock.unlock();

            notifyServerStatusChangeListener(changedServers);
        } finally {
            pingInProgress.set(false);
        }
    }

    ```
    主要是把所有server都ping一遍，然后重新设置upServerList,把down掉的server剔除，加上已经恢复的server。

#### DynamicServerListLoadBalancer

>  该类继承于BaseLoadBalancer类，它是对基础负载均衡器的扩展。在该负载均衡器，实现了服务实例清单在运行期的动态更新能力；同时，它还具备了对服务实例清单的过滤功能。

```java
    protected AtomicBoolean serverListUpdateInProgress = new AtomicBoolean(false);

    volatile ServerList<T> serverListImpl;

    volatile ServerListFilter<T> filter;

    protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
        @Override
        public void doUpdate() {
            updateListOfServers();
        }
    };

    protected volatile ServerListUpdater serverListUpdater;
```

负载均衡器中增加了成员变量有ServerList，ServerListFilter，UpdateAction,ServerListUpdater.

在DynamicServerListLoadBalancer的构造方法中

1. ServerList

    在DynamicServerListLoadBalancer的构造方法中调用了这个方法用来更新serverList：
    
    ```java
    @VisibleForTesting
    public void updateListOfServers() {
        List<T> servers = new ArrayList<T>();
        if (serverListImpl != null) {
            servers = serverListImpl.getUpdatedListOfServers();
            LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);

            if (filter != null) {
                servers = filter.getFilteredListOfServers(servers);
                LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                        getIdentifier(), servers);
            }
        }
        updateAllServerList(servers);
    }
    ```

    ```java
    public interface ServerList<T extends Server> {
    
        public List<T> getInitialListOfServers();
        
        /**
         * Return updated list of servers. This is called say every 30 secs
         * (configurable) by the Loadbalancer's Ping cycle
         * 
         */
        public List<T> getUpdatedListOfServers();   
    
    }
    ```
    通过查找配置类可以发现，这个类的实例是由DomainExtractingServerList实例化的。
    
    ```java
    @Bean
    @ConditionalOnMissingBean
    public ServerList<?> ribbonServerList(IClientConfig config, Provider<EurekaClient> eurekaClientProvider) {
        if (this.propertiesFactory.isSet(ServerList.class, serviceId)) {
            return this.propertiesFactory.get(ServerList.class, config, serviceId);
        }
        DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(
                config, eurekaClientProvider);
        DomainExtractingServerList serverList = new DomainExtractingServerList(
                discoveryServerList, config, this.approximateZoneFromHostname);
        return serverList;
    }
    ```
    注意DomainExtractingServerList的构造函数，传入了一个DiscoveryEnabledNIWSServerList discoveryServerList，DomainExtractingServerList对接口的具体实现就调用了这个传入参数的的对应方法。看一下DiscoveryEnabledNIWSServerList的具体实现
    ```java
    @Override
    public List<DiscoveryEnabledServer> getInitialListOfServers(){
        return obtainServersViaDiscovery();
    }

    @Override
    public List<DiscoveryEnabledServer> getUpdatedListOfServers(){
        return obtainServersViaDiscovery();
    }
    ```
    这高端操作，我一直没看明白这两个方法为什么最终居然是一个实现，我还以为这不符合代码规范呢。没想到啊，Spring这浓眉大眼的家伙也叛变革命了！
    
    ```java
        private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
            List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();
    
            if (eurekaClientProvider == null || eurekaClientProvider.get() == null) {
                logger.warn("EurekaClient has not been initialized yet, returning an empty list");
                return new ArrayList<DiscoveryEnabledServer>();
            }
    
            EurekaClient eurekaClient = eurekaClientProvider.get();
            if (vipAddresses!=null){
                for (String vipAddress : vipAddresses.split(",")) {
                    // if targetRegion is null, it will be interpreted as the same region of client
                    List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
                    for (InstanceInfo ii : listOfInstanceInfo) {
                        if (ii.getStatus().equals(InstanceStatus.UP)) {
    
                            if(shouldUseOverridePort){
                                if(logger.isDebugEnabled()){
                                    logger.debug("Overriding port on client name: " + clientName + " to " + overridePort);
                                }
    
                                // copy is necessary since the InstanceInfo builder just uses the original reference,
                                // and we don't want to corrupt the global eureka copy of the object which may be
                                // used by other clients in our system
                                InstanceInfo copy = new InstanceInfo(ii);
    
                                if(isSecure){
                                    ii = new InstanceInfo.Builder(copy).setSecurePort(overridePort).build();
                                }else{
                                    ii = new InstanceInfo.Builder(copy).setPort(overridePort).build();
                                }
                            }
    
                            DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, isSecure, shouldUseIpAddr);
                            des.setZone(DiscoveryClient.getZone(ii));
                            serverList.add(des);
                        }
                    }
                    if (serverList.size()>0 && prioritizeVipAddressBasedServers){
                        break; // if the current vipAddress has servers, we dont use subsequent vipAddress based servers
                    }
                }
            }
            return serverList;
        }
    ```
    EurekaClient通过服务名来返回对应的`List<InstanceInfo>`,然后把InstanceInfo转换成DiscoveryEnabledServer返回，DomainExtractingServerList会做后一步的处理。
    
    ```java
    	private List<DiscoveryEnabledServer> setZones(List<DiscoveryEnabledServer> servers) {
    		List<DiscoveryEnabledServer> result = new ArrayList<>();
    		boolean isSecure = this.ribbon.isSecure(true);
    		boolean shouldUseIpAddr = this.ribbon.isUseIPAddrForServer();
    		for (DiscoveryEnabledServer server : servers) {
    			result.add(new DomainExtractingServer(server, isSecure, shouldUseIpAddr,
    					this.approximateZoneFromHostname));
    		}
    		return result;
    	}
    ```
2. ServerListUpdater

    构造方法中，同时调用了
    ```java
        public void enableAndInitLearnNewServersFeature() {
            LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
            serverListUpdater.start(updateAction);
        }
    ```

    ```java
    public interface ServerListUpdater {
    
        /**
         * an interface for the updateAction that actually executes a server list update
         */
        public interface UpdateAction {
            void doUpdate();
        }
    
    
        /**
         * start the serverList updater with the given update action
         * This call should be idempotent.
         *
         * @param updateAction
         */
        void start(UpdateAction updateAction);
    
        /**
         * stop the serverList updater. This call should be idempotent
         */
        void stop();
    
        /**
         * @return the last update timestamp as a {@link java.util.Date} string
         */
        String getLastUpdate();
    
        /**
         * @return the number of ms that has elapsed since last update
         */
        long getDurationSinceLastUpdateMs();
    
        /**
         * @return the number of update cycles missed, if valid
         */
        int getNumberMissedCycles();
    
        /**
         * @return the number of threads used, if vaid
         */
        int getCoreThreads();
    }

    ```
    这个接口没有什么可说的，重点在它的实现类中，它的默认实现是PollingServerListUpdater
    ```java
        @Override
        public synchronized void start(final UpdateAction updateAction) {
            if (isActive.compareAndSet(false, true)) {
                final Runnable wrapperRunnable = new Runnable() {
                    @Override
                    public void run() {
                        if (!isActive.get()) {
                            if (scheduledFuture != null) {
                                scheduledFuture.cancel(true);
                            }
                            return;
                        }
                        try {
                            updateAction.doUpdate();
                            lastUpdated = System.currentTimeMillis();
                        } catch (Exception e) {
                            logger.warn("Failed one update cycle", e);
                        }
                    }
                };
    
                scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                        wrapperRunnable,
                        initialDelayMs,
                        refreshIntervalMs,
                        TimeUnit.MILLISECONDS
                );
            } else {
                logger.info("Already active, no-op");
            }
        }
    ```
    这个start方法的实现中可以看出来，启动了一个定时任务按照延迟1s后开始，每30s重复执行来更新serverList。
    
3. ServerListFilter
    
    前文可以看到serverList更新之后并不是直接返回，而是继续调用filter来对Eureka返回的结果进行进一步的过滤。
    
    ```java
    public interface ServerListFilter<T extends Server> {
    
        public List<T> getFilteredListOfServers(List<T> servers);
    
    }
    ```
    这个接口的实现在本文中就像一个递归，仍然是一个接口，然后抽象类，然后base，然后再具体的实现。
    
    在抽象的这一层AbstractServerListFilter增加了一个成员变量LoadBalancerStats，这是一个老朋友了，它提供了server的统计信息。
   
    3.1 ZoneAffinityServerListFilter 这就相当于base了
    >ZoneAffinityServerListFilter：该过滤器基于“区域感知（Zone Affinity）”的方式实现服务实例的过滤，也就是说它会根据提供服务的实例所处区域（Zone）与消费者自身的所处区域（Zone）进行比较，过滤掉那些不是同处一个区域的实例。
    
    ```java
    @Override
    public List<T> getFilteredListOfServers(List<T> servers) {
        if (zone != null && (zoneAffinity || zoneExclusive) && servers !=null && servers.size() > 0){
            List<T> filteredServers = Lists.newArrayList(Iterables.filter(
                    servers, this.zoneAffinityPredicate.getServerOnlyPredicate()));
            if (shouldEnableZoneAffinity(filteredServers)) {
                return filteredServers;
            } else if (zoneAffinity) {
                overrideCounter.increment();
            }
        }
        return servers;
    }
    ```
    可以看到这里通过Iterables来进行过滤，过滤条件是第二个参数一个Predicate类，这个参数在ZoneAffinityServerListFilter中指定为ZoneAffinityPredicate
    ```java
        private ZoneAffinityPredicate zoneAffinityPredicate = new ZoneAffinityPredicate();
    ```
    这样通过指定过滤条件为后续拓展提供无限可能。事实上后续的其他Filter都有一个对应的Predicate实现。
    
    ```java
    @Override
    public boolean apply(PredicateKey input) {
        Server s = input.getServer();
        String az = s.getZone();
        if (az != null && zone != null && az.toLowerCase().equals(zone.toLowerCase())) {
            return true;
        } else {
            return false;
        }
    }
    ```
    zoneAffinityPredicate主要根据zone来将不同的zone过滤出去。
    
    ```java
        private boolean shouldEnableZoneAffinity(List<T> filtered) {    
            if (!zoneAffinity && !zoneExclusive) {
                return false;
            }
            if (zoneExclusive) {
                return true;
            }
            LoadBalancerStats stats = getLoadBalancerStats();
            if (stats == null) {
                return zoneAffinity;
            } else {
                logger.debug("Determining if zone affinity should be enabled with given server list: {}", filtered);
                ZoneSnapshot snapshot = stats.getZoneSnapshot(filtered);
                double loadPerServer = snapshot.getLoadPerServer();
                int instanceCount = snapshot.getInstanceCount();            
                int circuitBreakerTrippedCount = snapshot.getCircuitTrippedCount();
                if (((double) circuitBreakerTrippedCount) / instanceCount >= blackOutServerPercentageThreshold.get() 
                        || loadPerServer >= activeReqeustsPerServerThreshold.get()
                        || (instanceCount - circuitBreakerTrippedCount) < availableServersThreshold.get()) {
                    logger.debug("zoneAffinity is overriden. blackOutServerPercentage: {}, activeReqeustsPerServer: {}, availableServers: {}", 
                            new Object[] {(double) circuitBreakerTrippedCount / instanceCount,  loadPerServer, instanceCount - circuitBreakerTrippedCount});
                    return false;
                } else {
                    return true;
                }
                
            }
        }
            
    ```
    调用后的结果并不一定有用，还是通过了shouldEnableZoneAffinity来进一步的判断是返回过滤结果还是原样返回，判断标准是过滤后的server：
    
    > blackOutServerPercentage：故障实例百分比（断路器断开数 / 实例数量） >= 0.8
    > activeReqeustsPerServer：实例平均负载 >= 0.6
    > availableServers：可用实例数（实例数量 - 断路器断开数） < 2
    
    其他Filter我就不一一写了，直接引用现成的结论好了。没有再自己去翻一遍源码的必要了。
    
    3.2 DefaultNIWSServerListFilter 继承前者，没有任何创新
    
    3.3 ServerListSubsetFilter：该过滤器也继承自ZoneAffinityServerListFilter，它非常适用于拥有大规模服务器集群(上百或更多)的系统。因为它可以产生一个“区域感知”结果的子集列表，同时它还能够通过比较服务实例的通信失败数量和并发连接数来判定该服务是否健康来选择性的从服务实例列表中剔除那些相对不够健康的实例。
        