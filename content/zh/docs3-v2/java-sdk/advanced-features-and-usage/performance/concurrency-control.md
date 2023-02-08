---
type: docs
title: "并发控制"
linkTitle: "并发控制"
weight: 28
description: "Dubbo 中的并发控制"
---
## 特性说明
多种并发控制功能，帮助用户管理其应用程序和服务。

## 使用场景
限制从同一客户端到同一服务的并发请求数，防止恶意请求使服务器过载，确保服务的稳定性，并防止使用过多资源。

控制某些服务的最大并发请求数，确保其他服务的资源可用性。系统过载和确保系统稳定性。

允许在需求增加时更平滑地扩展服务。

确保服务在高峰使用时间保持可靠和稳定。

## 使用方式
### 样例一

> 限制 `com.foo.BarService` 的每个方法，服务器端并发执行（或占用线程池线程数）不能超过 10 个

```xml
<dubbo:service interface="com.foo.BarService" executes="10" />
```

### 样例二

> 限制 `com.foo.BarService` 的 `sayHello` 方法，服务器端并发执行（或占用线程池线程数）不能超过 10 个

```xml
<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" executes="10" />
</dubbo:service>
```
### 样例三

> 限制 `com.foo.BarService` 的每个方法，每客户端并发执行（或占用连接的请求数）不能超过 10 个

```xml
<dubbo:service interface="com.foo.BarService" actives="10" />
```

**或**

```xml
<dubbo:reference interface="com.foo.BarService" actives="10" />
```

### 样例四

> 限制 `com.foo.BarService` 的 `sayHello` 方法，每客户端并发执行（或占用连接的请求数）不能超过 10 个

```xml
<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>
```

或

```xml
<dubbo:reference interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>
```

> 如果 `<dubbo:service>` 和 `<dubbo:reference>` 都配了actives，`<dubbo:reference>` 优先，参见：[配置的覆盖策略](../../../reference-manual/config/principle/)。

### Load Balance 均衡

配置服务的客户端的 `loadbalance` 属性为 `leastactive`，此 Loadbalance 会调用并发数最小的 Provider（Consumer端并发数）。

```xml
<dubbo:reference interface="com.foo.BarService" loadbalance="leastactive" />
```

**或**

```xml
<dubbo:service interface="com.foo.BarService" loadbalance="leastactive" />
```

## 自适应限流
从理论上讲，服务端机器的处理能力是存在上限的，对于一台服务端机器，当短时间内出现大量的请求调用的时候，会导致处理不及时的请求积压，使机器过载。在这种情况下可能导致两个问题：1.由于请求积压，最终所有的请求都必须经过较长时间的等待才会被处理，从而导致整个服务瘫痪。2.服务端长时间的过载可能导致宕机。因此，在存在过载风险的时候，拒绝掉一部分请求反而是更好的选择。而之前的静态的并发控制需要静态的设置最大并发值，但是在复杂情况下该值难以合理的确定。基于以上原因，我们需要一种自适应的方法，其可以动态调整服务端机器的最大并发值，使其可以在保证不过载的前提下，尽可能多的处理接收到的请求。

### HeuristicSmoothingFlowControl
当服务端收到一个请求时候，首先判断CPU的使用率是否超过50%。如果超过则说明当前负载较高，便通过HeuristicSmoothingFlowControl算法中获得当前的maxConcurrency值。如果当前正在处理的请求数量超过了maxConcurrency的值，则拒绝该请求。

*maxConcurrency = ceil(maxQPS*((2+alpha) * noLoadLatency - avgLatency))

### AutoConcurrencyLimier
AutoConcurrencyLimier 的算法使用过程和 HeuristicSmoothingFlowControl类似。最大的区别是AutoConcurrencyLimier是基于窗口的。每当窗口内积累了一定量的采集数据时，才利用窗口内的数据来更新maxConcurrency。其次，利用exploreRatio来对剩余的容量进行探索。另外，每隔一段时间都会自动缩小max_concurrency并持续一段时间，以处理onLoadLatency上涨的情况。因为估计noLoadLatency时必须先让服务处于低负载的状态，因此对maxConcurrency的缩小是难以避免的。由于maxConcurrency<concurrency时，服务器会拒绝掉所有的请求，限流算法将 ”排空所有的经历过排队的等待请求的时间“设置为 2 * latency，以确保minLatency的样本绝大部分没有经过排队等待。

### 样例1
```xml
<dubbo:reference interface="com.foo.BarService" flowcontrol="heuristicSmoothingFlowControl" />
```

### 样例2
```xml
<dubbo:reference interface="com.foo.BarService" flowcontrol="autoConcurrencyLimier" />
```

