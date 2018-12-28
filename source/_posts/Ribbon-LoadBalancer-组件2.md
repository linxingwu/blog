---
title: Ribbon LoadBalancer 组件2
date: 2018-12-27 16:31:54
tags:
 - Spring Cloud
 - Ribbon
 - ZoneAwareLoadBalancer
---
接着上一篇，继续说说 Ribbon 的LoadBalancer

# ZoneAwareLoadBalancer

 这个类继承了前文中的DynamicServerListLoadBalancer，避免轮询时候跨zone调用实体产生高延迟。主要通过重写了两个方法：
 
 ## setServerListForZones
 ```java
         private ConcurrentHashMap<String, BaseLoadBalancer> balancers = new ConcurrentHashMap<String, BaseLoadBalancer>();
         
         @Override
         protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
             super.setServerListForZones(zoneServersMap);
             if (balancers == null) {
                 balancers = new ConcurrentHashMap<String, BaseLoadBalancer>();
             }
             for (Map.Entry<String, List<Server>> entry: zoneServersMap.entrySet()) {
                 String zone = entry.getKey().toLowerCase();
                 getLoadBalancer(zone).setServersList(entry.getValue());
             }
             // check if there is any zone that no longer has a server
             // and set the list to empty so that the zone related metrics does not
             // contain stale data
             for (Map.Entry<String, BaseLoadBalancer> existingLBEntry: balancers.entrySet()) {
                 if (!zoneServersMap.keySet().contains(existingLBEntry.getKey())) {
                     existingLBEntry.getValue().setServersList(Collections.emptyList());
                 }
             }
         }   
 ```
 第一个方法创建了一个map，为每一个zone配置一个LoadBalancer,初始情况zone没有对应的LoadBalancer，则调用getLoadBalancer来为zone设置一个。
 
     ```java
        BaseLoadBalancer getLoadBalancer(String zone) {
            zone = zone.toLowerCase();
            BaseLoadBalancer loadBalancer = balancers.get(zone);
            if (loadBalancer == null) {
                // We need to create rule object for load balancer for each zone
                IRule rule = cloneRule(this.getRule());
                loadBalancer = new BaseLoadBalancer(this.getName() + "_" + zone, rule, this.getLoadBalancerStats());
                BaseLoadBalancer prev = balancers.putIfAbsent(zone, loadBalancer);
                if (prev != null) {
                    loadBalancer = prev;
                }
            } 
            return loadBalancer;        
        }
    ```

 这个方法简单明了，有Rule就根据config和Rule的class实例化一个，没有就new AvailabilityFilteringRule()。然后new一个BaseLoadBalancer，最后把新的LoadBalancer放到map里。随后在chooseServer的时候发挥作用。在RibbonClientConfiguration中可以得知，最终的默认实现是ZoneAvoidanceRule。
 
 ## chooseServer
 
 ```java        
    @Override
    public Server chooseServer(Object key) {
        if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
            logger.debug("Zone aware logic disabled or there is only one zone");
            return super.chooseServer(key);
        }
        Server server = null;
        try {
            LoadBalancerStats lbStats = getLoadBalancerStats();
            Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
            logger.debug("Zone snapshots: {}", zoneSnapshot);
            if (triggeringLoad == null) {
                triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                        "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
            }
    
            if (triggeringBlackoutPercentage == null) {
                triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                        "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
            }
            Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
            logger.debug("Available zones: {}", availableZones);
            if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
                String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
                logger.debug("Zone chosen: {}", zone);
                if (zone != null) {
                    BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                    server = zoneLoadBalancer.chooseServer(key);
                }
            }
        } catch (Exception e) {
            logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
        }
        if (server != null) {
            return server;
        } else {
            logger.debug("Zone avoidance logic is not invoked.");
            return super.chooseServer(key);
        }
    }
 ```
 可以看到chooseServer的时候先判断是否要根据区域来进行选择，如果配置了否或者zone只有一个就直接用父类轮询的策略选择。
 
 然后根据LoadBalancerStats创建zoneSnapshot，保存了每个zone的统计信息，包括断路器触发数量，平均负载等信息。然后读取了两个参数，一个是故障比例0.99，一个是服务负载下限0.2。调用ZoneAvoidanceRule.getAvailableZones方法。
 
 ## 委托父类
 
 ```java
    public static Set<String> getAvailableZones(
            Map<String, ZoneSnapshot> snapshot, double triggeringLoad,
            double triggeringBlackoutPercentage) {
        if (snapshot.isEmpty()) {
            return null;
        }
        Set<String> availableZones = new HashSet<String>(snapshot.keySet());
        if (availableZones.size() == 1) {
            return availableZones;
        }
        Set<String> worstZones = new HashSet<String>();
        double maxLoadPerServer = 0;
        boolean limitedZoneAvailability = false;

        for (Map.Entry<String, ZoneSnapshot> zoneEntry : snapshot.entrySet()) {
            String zone = zoneEntry.getKey();
            ZoneSnapshot zoneSnapshot = zoneEntry.getValue();
            int instanceCount = zoneSnapshot.getInstanceCount();
            if (instanceCount == 0) {
                availableZones.remove(zone);
                limitedZoneAvailability = true;
            } else {
                double loadPerServer = zoneSnapshot.getLoadPerServer();
                if (((double) zoneSnapshot.getCircuitTrippedCount())
                        / instanceCount >= triggeringBlackoutPercentage
                        || loadPerServer < 0) {
                    availableZones.remove(zone);
                    limitedZoneAvailability = true;
                } else {
                    if (Math.abs(loadPerServer - maxLoadPerServer) < 0.000001d) {
                        // they are the same considering double calculation
                        // round error
                        worstZones.add(zone);
                    } else if (loadPerServer > maxLoadPerServer) {
                        maxLoadPerServer = loadPerServer;
                        worstZones.clear();
                        worstZones.add(zone);
                    }
                }
            }
        }

        if (maxLoadPerServer < triggeringLoad && !limitedZoneAvailability) {
            // zone override is not needed here
            return availableZones;
        }
        String zoneToAvoid = randomChooseZone(snapshot, worstZones);
        if (zoneToAvoid != null) {
            availableZones.remove(zoneToAvoid);
        }
        return availableZones;

    }
 ```
 getAvailableZones方法内部创建了两个set，一个用于保存候选zone，一个用来保存最差的zone，最差指的是负载最高的。候选zone的实例数量必须大于0，而且（断路器处罚数量/实例数）不能超过0.99。
 
 如果最高负载比传入的最低负载还要低，所有实例都符合候选条件，就直接返回，大家都可用。不然就在最差区域中随机选一个zone移除掉，返回剩下的候选zone。
 
  getAvailableZones返回后，ZoneAwareLoadBalancer继续判断，如果剔除过zone，并且可用zone不为空，随机选择一个zone，委托这个zone对应的LoadBalancer来进行下一步的选择，很明显这里最终会委托给IRule来进行选择，这个具体的Rule，应该是父类的RoundRobinRule，或者这里新设置的AvailabilityFilteringRule。书上说的是ZoneAvoidanceRule来实现。可能是我有关键信息没找到，不知道怎么会是这个Rule。
  
  这样最终就能实现根据zone来选择服务实例的效果。