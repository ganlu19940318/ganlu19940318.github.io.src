---
title: Spring Cloud微服务(三)--客户端负载均衡Spring Cloud Ribbon
date: 2018-11-23 12:27:26
categories: Spring Cloud
tags: [Spring Cloud, 微服务, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是 << Spring Cloud微服务实战 >> 学习笔记, 以便自己查阅.

Spring Cloud Ribbon是一个基于HTTP和TCP的客户端负载均衡工具,它基于Netflix Ribbon实现.

# 2. 客户端负载均衡

负载均衡是对系统的高可用,网络压力的缓解和处理内容扩容的重要手段之一.
负载均衡可以分为客户端负载均衡和服务端负载均衡.
负载均衡按设备来分为硬件负载均衡和软件负载均衡,都属于服务端负载均衡.

硬件负载均衡主要通过在服务器节点之间安装专门用于负载均衡的设备,例如F5等.
软件负载均衡通过在服务器上安装一些具有负载均衡功能或模块的软件来完成请求的转发工作,例如Nginx等.

硬件负载均衡和软件负载均衡都会维护一个可用的服务清单,然后通过心跳检测来剔除故障节点以保证服务清单中的节点都正常可用.当客户端发出请求时,负载均衡器会按照某种算法(线性轮询,按权重负载,按流量负载等)从服务清单中取出一台服务器的地址,然后将请求转发到该服务器上.

客户端负载均衡需要客户端自己维护自己要访问的服务实例清单,这些服务清单来源于注册中心(在使用Eureka进行服务治理时).

服务端负载均衡架构方式
![服务端负载均衡](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/897287-20170526002234638-1191131982.png)

# 3. 基于Spring Cloud Ribbon实现客户端负载均衡

基于Spring Cloud Ribbon实现客户端负载均衡非常简单,主要由以下步骤:

1. 服务提供者需要启动多个服务实例并注册到一个或多个相关联的服务注册中心上;

2. 服务消费者直接通过带有@LoadBalanced注解的RestTemplate向服务提供者发送请求以实现客户端的负载均衡.

# 4. 原理

使用被@LoadBalanced注解的RestTemplate发起请求时,会被LoadBalancerInterceptor拦截,然后借助负载均衡器LoadBalancerClient将逻辑服务名转换为host:port的具体的服务实例地址,在使用RibbonLoadBalancerClient(Ribbon实现的负载均衡器)时实际使用的是Ribbon中定义的ILoadBalancer,默认自动化配置的负载均衡器是ZoneAwareLoadBalancer.

# 5. 负载均衡策略

这一部分主要引自外部, 我觉得负载均衡是理解这块必不可少的, 可以提前了解一下, 但是实际使用中, 应该还是得根据业务需求选择最合适的. 并且选择之后, 还是得看对应的源码部分, 以便了解这些策略下可能会出现什么问题. 所以这里的源码解读只作参考.

## 5.1 RandomRule

该策略实现了从服务实例清单中随机选择一个服务实例的功能.下面先看一下源码:

```java
package com.netflix.loadbalancer;

import java.util.List;
import java.util.Random;

import com.netflix.client.config.IClientConfig;

public class RandomRule extends AbstractLoadBalancerRule {
    Random rand;

    public RandomRule() {
        rand = new Random();
    }

    @edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "RCN_REDUNDANT_NULLCHECK_OF_NULL_VALUE")
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            if (Thread.interrupted()) {
                return null;
            }
            List<Server> upList = lb.getReachableServers();
            List<Server> allList = lb.getAllServers();

            int serverCount = allList.size();
            if (serverCount == 0) {
                /*
                 * No servers. End regardless of pass, because subsequent passes
                 * only get more restrictive.
                 */
                return null;
            }

            int index = rand.nextInt(serverCount);
            server = upList.get(index);

            if (server == null) {
                /*
                 * The only time this should happen is if the server list were
                 * somehow trimmed. This is a transient condition. Retry after
                 * yielding.
                 */
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Shouldn't actually happen.. but must be transient or a bug.
            server = null;
            Thread.yield();
        }

        return server;

    }

    @Override
    public Server choose(Object key) {
        return choose(getLoadBalancer(), key);
    }

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        // TODO Auto-generated method stub
    }
}
```

分析源码可以看出,IRule接口中Server choose(Object key)函数的实现委托给了该类中的Server choose(ILoadBalancer lb, Object key)函数,该方法增加了一个负载均衡器参数.从具体的实现可以看出,它会使用负载均衡器来获得可用实例列表upList和所有的实例列表allList,并且使用rand.nextInt(serverCount)函数来获取一个随机数,并将该随机数作为upList的索引值来返回具体实例.同时,具体的选择逻辑在一个while(server == null)循环之内,而根据选择逻辑的实现,正常情况下每次都应该选出一个服务实例.

## 5.2 RoundRobinRule

该策略实现了按照线性轮询的方式依次选择每个服务实例的功能.下面看一下源码:

```java
package com.netflix.loadbalancer;

import com.netflix.client.config.IClientConfig;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class RoundRobinRule extends AbstractLoadBalancerRule {
    private AtomicInteger nextServerCyclicCounter;

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
    /**
     * Inspired by the implementation of {@link AtomicInteger#incrementAndGet()}.
     *
     * @param modulo The modulo to bound the value of the counter.
     * @return The next value.
     */
    private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextServerCyclicCounter.get();
            int next = (current + 1) % modulo;
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }
}
```

RoundRobinRule具体实现和RandomRule类似,但是循环条件和从可用列表获取实例的逻辑不同.循环条件中增加了一个count计数变量,该变量会在每次循环之后累加,如果循环10次还没获取到Server,就会结束,并打印一个警告信息No available alive servers after 10 tries from load balancer:...

线性轮询的实现是通过AtomicInteger nextServerCyclicCounter对象实现,每次进行实例选择时通过调用int incrementAndGetModulo(int modulo)方法来实现.

## 5.3 RetryRule

该策略实现了一个具备重试机制的实例选择功能. 从源码中可以看出, 内部定义了一个IRule对象, 默认是RoundRobinRule实例, choose方法中则实现了对内部定义的策略进行反复尝试的策略, 若期间能够选择到具体的服务实例就返回, 若选择不到并且超过设置的尝试结束时间(maxRetryMillis参数定义的值 + choose方法开始执行的时间戳)就返回null.

```java
package com.netflix.loadbalancer;

import com.netflix.client.config.IClientConfig;

public class RetryRule extends AbstractLoadBalancerRule {
    IRule subRule = new RoundRobinRule();
    long maxRetryMillis = 500;

    /*
     * Loop if necessary. Note that the time CAN be exceeded depending on the
     * subRule, because we're not spawning additional threads and returning
     * early.
     */
    public Server choose(ILoadBalancer lb, Object key) {
        long requestTime = System.currentTimeMillis();
        long deadline = requestTime + maxRetryMillis;

        Server answer = null;

        answer = subRule.choose(key);

        if (((answer == null) || (!answer.isAlive()))
                && (System.currentTimeMillis() < deadline)) {

            InterruptTask task = new InterruptTask(deadline
                    - System.currentTimeMillis());

            while (!Thread.interrupted()) {
                answer = subRule.choose(key);

                if (((answer == null) || (!answer.isAlive()))
                        && (System.currentTimeMillis() < deadline)) {
                    /* pause and retry hoping it's transient */
                    Thread.yield();
                } else {
                    break;
                }
            }

            task.cancel();
        }

        if ((answer == null) || (!answer.isAlive())) {
            return null;
        } else {
            return answer;
        }
    }

}
```

## 5.4 WeightedResponseTimeRule

该策略是对RoundRobinRule的扩展, 增加了根据实例的运行情况来计算权重, 并根据权重来挑选实例, 以达到更优的分配效果. 它的实现主要有三个核心内容.

### 5.4.1 定时任务

WeightedResponseTimeRule策略在初始化的时候会通过serverWeightTimer.schedule(new DynamicServerWeightTask(), 0, serverWeightTaskTimerInterval)启动一个定时任务, 用来为每个服务实例计算权重,该任务默认30s执行一次.

### 5.4.2 权重计算

在源码中我们可以轻松找到用于存储权重的对象private volatile List < Double > accumulatedWeights = new ArrayList<Double>();该List中每个权重值所处的位置对应了负载均衡器维护的服务实例清单中所有实例在清单中的位置. 下面看一下权重计算函数maintainWeights的源码:

```java
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

该方法的实现主要分为两个步骤:

根据LoadBalancerStats中记录的每个实例的统计信息,累加所有实例的平均响应时间,得到总平均响应时间totalResponseTime,该值会用于后续的计算.
为负载均衡器中维护的实例清单逐个计算权重(从第一个开始),计算规则为weightSoFar + totalResponseTime - 实例的平均响应时间,其中weightSoFar初始化为0,并且每计算好一个权重需要累加到weightSoFar上供下一次计算使用.
通过概算计算出来的权重值只是代表了各实例权重区间的上限.下面图节选自<< Spring Cloud微服务实战 >>.

![Spring Cloud微服务实战](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/1556759493-5b74c261f0624_articlex.png)

### 5.4.3 实例选择

下面看一下Server choose(ILoadBalancer lb, Object key)如何选择Server的.

```java
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

下面我们看一下源码的主要步骤有：

首先先获取accumulatedWeights中最后一个权重,如果该权重小于0.001或者实例的数量不等于权重列表的数量,就采用父类的线性轮询策略.
如果满足条件,就先产生一个[0,最大权重值)区间内的随机数.
遍历权重列表,比较权重值与随机数的大小,如果权重值大于等于随机数,就拿当前权重列表的索引值去服务实例列表获取具体的实例.

## 5.5 ClientConfigEnabledRoundRobinRule

该策略比较特殊,一般不直接使用它.因为他本身并没有实现特殊的处理逻辑,在他内部定义了一个RoundRobinRule策略,choose函数的实现其实就是采用了RoundRobinRule的线性轮询机制.

在实际开发中,我们并不会直接使用该策略,而是基于它做高级策略扩展.

## 5.6 BestAvailableRule

该策略继承自ClientConfigEnabledRoundRobinRule,在实现中它注入了负载均衡器的统计对象LoadBalancerStats,同时在choose方法中利用LoadBalancerStats保存的实例统计信息来选择满足要求的服务实例.

当LoadBalancerStats为空时,会使用RoundRobinRule线性轮询策略,当有LoadBalancerStats时,会通过遍历负载均衡器中维护的所有服务实例,会过滤掉故障的实例,并找出并发请求数最小的一个.

该策略的特性是可以选出最空闲的服务实例.

## 5.7 PredicateBasedRule

这是一个抽象策略,它继承了ClientConfigEnabledRoundRobinRule,从命名中可以猜出这是一个基于Predicate实现的策略,Predicate是Google Guava Collection工具对集合进行过滤的条件接口.

```java
    public Server choose(Object key) {
        ILoadBalancer lb = getLoadBalancer();
        Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
        if (server.isPresent()) {
            return server.get();
        } else {
            return null;
        }
    }
```

在该源码中,它定义了一个抽象函数getPredicate来获取AbstractServerPredicate对象的实现,在choose方法中,通过AbstractServerPredicate的chooseRoundRobinAfterFiltering函数来选择具体的服务实例.从该方法的命名我们可以看出大致的逻辑:首先通过子类中实现的Predicate逻辑来过滤一部分服务实例,然后再以线性轮询的方式从过滤后的实例清单中选出一个.

在上面choose函数中调用的chooseRoundRobinAfterFiltering方法先通过内部定义的getEligibleServers函数来获取备选的实例清单(实现了过滤),如果返回的清单为空,则用Optional.absent来表示不存在,反之则以线性轮询的方式从备选清单中获取一个实例.

下面看一下getEligibleServers方法的源码:

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

上述源码的大致逻辑是遍历服务清单,使用this.apply方法来判断实例是否需要保留,如果是就添加到结果列表中.

实际上,AbstractServerPredicate实现了com.google.common.base.Predicate接口,apply方法是接口中的定义,主要用来实现过滤条件的判断逻辑,它输入的参数则是过滤条件需要用到的一些信息(比如源码中的new PredicateKey(loadBalancerKey, server)),传入了关于实例的统计信息和负载均衡器的选择算法传递过来的key.

AbstractServerPredicate没有apply的实现,所以这里的chooseRoundRobinAfterFiltering方法只是定义了一个模板策略:先过滤清单,再轮询选择.

对于如何过滤,需要在AbstractServerPredicate的子类中实现apply方法来确定具体的过滤策略.

## 5.8 AvailabilityFilteringRule

该类继承自PredicateBasedRule,遵循了先过滤清单,再轮询选择的基本处理逻辑,其中过滤条件使用了AvailabilityPredicate,下面看一下AvailabilityPredicate的源码:

```java
package com.netflix.loadbalancer;

import javax.annotation.Nullable;

import com.netflix.client.config.IClientConfig;
import com.netflix.config.ChainedDynamicProperty;
import com.netflix.config.DynamicBooleanProperty;
import com.netflix.config.DynamicIntProperty;
import com.netflix.config.DynamicPropertyFactory;

public class AvailabilityPredicate extends  AbstractServerPredicate {
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

}
```

从上面的源码可以看出,主要过的过滤逻辑都是在boolean shouldSkipServer(ServerStats stats)方法中实现,该方法主要判断服务实例的两项内容:

是否故障,即断路由器是否生效已断开
实例的并发请求数大于阀值,默认值2^32 - 1,该配置可以通过参数来修改

上面两项只要满足一项,apply方法就返回false,代表该服务实例可能存在故障或负载过高,都不满足就返回true.

在AvailabilityFilteringRule进行实例选择时做了小小的优化,它并没有向父类一样先遍历所有的节点进行过滤,然后在过滤后的集合中选择实例.而是先以线性的方式选择一个实例,接着使用过滤条件来判断该实例是否满足要求,若满足就直接使用该实例,若不满足要求就再选择下一个实例,检查是否满足要求,这个过程循环10次如果还没有找到合适的服务实例,就采用父类的实现方案.

该策略通过线性轮询的方式直接尝试寻找可用且比较空闲的实例来用,优化了每次都要遍历所有实例的开销.

## 5.9 ZoneAvoidanceRule

该类也是PredicateBasedRule的子类,它的实现是通过组合过滤条件CompositePredicate,以ZoneAvoidancePredicate为主过滤条件,以AvailabilityPredicate为次过滤条件.

ZoneAvoidanceRule的实现并没有像AvailabilityFilteringRule重写choose函数来优化,所以它遵循了先过滤清单再轮询选择的基本逻辑.

下面看一下CompositePredicate的源码

```java

package com.netflix.loadbalancer;

import java.util.Iterator;
import java.util.List;

import javax.annotation.Nullable;

import com.google.common.base.Predicate;
import com.google.common.base.Predicates;
import com.google.common.collect.Lists;

public class CompositePredicate extends AbstractServerPredicate {

    private AbstractServerPredicate delegate;
    private List<AbstractServerPredicate> fallbacks = Lists.newArrayList();
    private int minimalFilteredServers = 1;
    private float minimalFilteredPercentage = 0;
    @Override
    public boolean apply(@Nullable PredicateKey input) {
        return delegate.apply(input);
    }
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
}
```

从源码中可以看出,CompositePredicate定义了一个主过滤条件delegate和一组过滤条件列表fallbacks,次过滤条件的过滤顺序是按存储顺序执行的.

在获取结果的getEligibleServers函数中的主要逻辑是:

使用主过滤条件对所有实例过滤并返回过滤后的实例清单
每次使用次过滤条件过滤前,都要判断两个条件,一个是过滤后的实例总数 >= 最小过滤实例数(minimalFilteredServers,默认值为1), 另一个是过滤后的实例比例 > 最小过滤百分比(minimalFilteredPercentage，默认为0),只要有一个不符合就不再进行过滤, 将当前服务实例列表返回
依次使用次过滤条件列表中的过滤条件对主过滤条件的过滤结果进行过滤.

# 6. 参考链接

<< Spring Cloud微服务实战 >>
[客户端负载均衡Spring Cloud Ribbon](https://segmentfault.com/a/1190000015981984)
[Spring Cloud Ribbon负载均衡策略](https://segmentfault.com/a/1190000016028992)