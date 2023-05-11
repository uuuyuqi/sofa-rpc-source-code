



### 说明

##### 1.核心入口 —— FailoverCluster

rpc 客户端发起调用时，调用的其实是客户端代理对象。代理对象调用请求的大致源码如下：

```java
SofaRequest request = MessageBuilder.buildSofaRequest(...);
SofaResponse response = this.proxyInvoker.invoke(request);
```

最终请求的发出，还是 proxyInvoker 委托 Cluster 对象做的。所以我们研究服务调用的源码，可以直接研究 FailoverCluster 类的源码。

> 怎么知道是 FailoverCluster 实现？可以在`rpc-config-default.json`配置文件中，看到这个默认配置：
>
> ```json
> "consumer.cluster": "failover
> ```
>
> 经过 sofa rpc 自定义的 spi 加载流程，就会使用 FailoverCluster 对象，其实 dubbo 默认也是 FailoverCluster。而 failover 是一个分布式系统理论中的概念，可以理解是：故障转移，即**出现故障，就换个节点再试试**。FailoverCluster 也很好理解，如果发生调用失败，就换个 provider 节点再次发起调用（前提是 consumer 端配置了重试机制）。
>



实际上，FailoverCluster 是 sofa rpc 中，consumer 端非常非常非常非常核心的类，它的作用类似于 dubbo 中的 FailoverClusterInvoker。是 consumer 发起请求这一操作的入口类。





#### 2.debug 技巧

使用 idea 进行 debug 时，默认会 hang 住所有线程，所以此时很难用 visualVM 等工具查看线程状态，而且比较要命的是，后台的 io 线程也是挂起的，导致 consumer 不会自动发心跳到 provider，从而被 provider 挂起，进而导致 debug 到最后莫名失败。所以建议 debug 时默认只去 hang 当前的线程。或者修改 provider 不摘除超时空闲连接。

```
bolt.tcp.heartbeat.switch
```

> #### 附——服务端空闲检测代码（from sofa-bolt）：
>
> ```java
> // rpc server 添加空闲检测
> if (System.getProperty("bolt.tcp.heartbeat.switch", "true")) {
>     pipeline.addLast("idleStateHandler", new IdleStateHandler(...));
>     pipeline.addLast("serverIdleHandler", serverIdleHandler);
> }
> 
> // rpc server 关闭空闲连接
> if (evt instanceof IdleStateEvent) {
>     try {
>         logger.warn(...);
>         ctx.close();
>     } catch (Exception e) {
>         ...
>     }
> }
> ```













### 〇、服务调用总览

#### 1.大致流程

在进行 rpc 服务调用时，一般会经过以下几个步骤：

- provider 选择
  - 排除失败节点
  - router 筛选
  - loadbalancer 筛选
- filterChain 处理
- 进行数据发送（末位 filterInvoker 委托 cluster 发起）
- 将数据提交给 transport 层进行发送
- transport 层委托底层通信模块（如 bolt）发送数据



#### 2.Failover Cluster 请求流程

我们请求调用的入口，就在 FailoverCluster#invoke 方法：

```java
@Override
public SofaResponse invoke(SofaRequest request) throws SofaRpcException {
    SofaResponse response = null;
    try {
        
        countOfInvoke.incrementAndGet(); // 优雅停机计数 +1
        // 发起服务调用
        response = doInvoke(request);
      
        return response;
    } catch (SofaRpcException e) {
        ...
    } finally {
        countOfInvoke.decrementAndGet(); // 计数 -1
    }
}
```

我们可以先大致浏览一下发起调用时进行的操作：

- 记录一下当前是第 n 次调用，放入到 rpc ctx 中，防止超出重试次数
- 记录一下前几次失败的总 provider 列表，以便本次调用时避开这些
- 如果调用出现了 **provider 端异常、或超时异常**，且 consumer 端要求失败重试，则继续重试剩余可用次数
- 如果调用过程中出现了**其他异常，则直接报错，跳过重试**（很好理解，业务异常没必要重试）

```java
@Override
public SofaResponse doInvoke(SofaRequest request) throws SofaRpcException {
    
    int retries = consumerConfig.getMethodRetries(...); // 配置的重试次数
    int time = 0; // 当前调用次数
    SofaRpcException throwable = null; // 异常
    List<ProviderInfo> invokedProviderInfos = new ArrayList<ProviderInfo>(retries + 1); // 总失败列表
    do {
        // 1.找出可用的 providers，2.选出本次要调用的 provider
        ProviderInfo providerInfo = select(request, invokedProviderInfos);
        try {
            // 3.经过 filterChain 过滤 4.发起调用并获得结果
            SofaResponse response = filterChain(providerInfo, request);
            if (response != null) {
                LOGGER.warn(...)
                return response;
            } else {
                throwable = ...;
                time++;
            }
        } catch (SofaRpcException e) { 
            // 服务端异常 + 超时异常 才发起rpc异常重试
            if (e.getErrorType() == RpcErrorType.SERVER_BUSY
                || e.getErrorType() == RpcErrorType.CLIENT_TIMEOUT) {
                throwable = e;
                time++;
            } else {
                throw e;
            }
        } catch (Exception e) { // 其它异常不重试
            throw e;
        } finally {
            time++
        }
        invokedProviderInfos.add(providerInfo);
    } while (time <= retries);

    throw throwable;
}
```

简化之后，核心操作就是：

```java
@Override
public SofaResponse doInvoke(SofaRequest request) throws SofaRpcException {
    do {
        // 选出 provider
        ProviderInfo providerInfo = select(request, invokedProviderInfos);
        // 过滤并调用
        SofaResponse response = filterChain(providerInfo, request);
    // 判断次数是否达到最大
    } while (time <= retries);
}
```







### 一、select provider

当 cluster 开始调用时，第一步则是找出可用的 provider 列表。而这个 provider 列表，就是来自注册中心客户端获取的数据。一般线上环境 provider 列表会包含多个节点：

```java
public SofaResponse doInvoke(SofaRequest request) throws SofaRpcException {
    ...
      
    ProviderInfo providerInfo = select(request, invokedProviderInfos);
    
}
```

select 方法是筛选 provider 的入口方法，里面主要包含 3 个主要操作：

- 经过 **router** 筛选可用 providers 列表
- 去除已经调用失败的节点
- 经过 l**oadBalancer** 筛选最终要调用的 provider

此外，select 方法中还会检测是否配置了 pinpoint（在调用上下文中配置的，一种内置的扩展点，可以理解是一种直连）：

```java
protected ProviderInfo select(SofaRequest message, List<ProviderInfo> invokedProviderInfos)
        throws SofaRpcException {
    ...
    
    
    // 1.经过 router 筛选可用 providers 列表
    List<ProviderInfo> providerInfos = routerChain.route(message, null);
  
    
    // 2.去除已经调用失败的节点
    providerInfos.removeAll(invokedProviderInfos);

    
    // 3.配置了 pinpoint 直连地址，则走直连
    if (StringUtils.isNotBlank(targetIP)) {
        providerInfo = selectPinpointProvider(targetIP, providerInfos);
        ClientTransport clientTransport = selectByProvider(message, providerInfo);
        if (clientTransport == null) {
            // 指定的不存在或已死，抛出异常
            throw unavailableProviderException(message.getTargetServiceUniqueName(), targetIP);
        }
        return providerInfo;
      
    // 4.没有配置直连，则对可用 provider 进行负载均衡
    } else {
        do {
            // 再进行负载均衡筛选
            providerInfo = loadBalancer.select(message, providerInfos);
            ClientTransport transport = selectByProvider(message, providerInfo);
            if (transport != null) {
                return providerInfo;
            }
            providerInfos.remove(providerInfo);
        } while (!providerInfos.isEmpty());
    }
    throw unavailableProviderException(message.getTargetServiceUniqueName(),
            convertProviders2Urls(originalProviderInfos));
}
```



#### 1.经过 Router 筛选 provider 列表

Router 可以认为是一种**路由选择器**，其作用可以对比参考 dubbo 的 **Directory** 类。目的提供可用的 provider 列表，这些 providers 信息经过 loadBalancer 筛选后就可以去调用了。一般我们常用的就是 RegistryRouter，顾名思义就是筛选出从注册中心（Registry）上获取的 provider 信息：

```java
List<ProviderInfo> providerInfos = routerChain.route(message, null);

public List<ProviderInfo> route(SofaRequest request, List<ProviderInfo> providerInfos) {
    for (Router router : routers) {
        // 每次 route 的结果都会放入到 providerInfos 中
        providerInfos = router.route(request, providerInfos);
    }
    return providerInfos;
}
```

Router 也是 sofa 中比较常用的扩展点，可以人为在此拓展添加额外的 provider 和制定一些路由规则。比如是否开启点对点直连，也是通过 router 实现的。直连的处理流程则是：先是 sofaboot 检测到 xml 配置了 target-url，或者注解中配置了 direct-url，就会把这些信息收集到 binding 参数中，然后在组装 consumerConfig 时，再把这个直连 url 信息 set 进去，等到加载 router 时，router 的 needToLoad 方法就会检测出，directUrl 不为空，则让 DirectUrlRouter 生效，让 RegistryRouter 不生效，从而实现单点路由。





#### 2.去除已经调用的节点

回顾一下，这个 FailoverCluster 是带有重试机制的调用器，在发生重试的情况下，我们应该记录已经失败的节点，并将其保存下来。在**下次调用时，主动避开这些失败的节点**，对其他节点调用。

```java
protected ProviderInfo select(SofaRequest message, List<ProviderInfo> invokedProviderInfos)
        throws SofaRpcException {
    ...
      
    if (CommonUtils.isNotEmpty(invokedProviderInfos) 
        && providerInfos.size() > invokedProviderInfos.size()) { // 总数大于已调用数
        providerInfos.removeAll(invokedProviderInfos);// 已经调用异常的本次不再重试
    }
  
    ...
  
}
```

但是此时有必要考虑一些边界情况：如果我总共只有 2 个节点，但是配置了 3 次重试，那避开了所有节点，还能调用什么呢？所以这也就是为什么每次去除失败节点之前，做一个判断 —— 去除失败数的节点后，是否还有可用节点了？如果没有了，那就退而求其次，对失败节点也进行调用。

> 这就是工程设计的哲学，不能被规则框死。在 sofastack 体系中，这样的设计还有很多。





#### 3.检查调用上下文中是否配置了 pinpoint

pinpoint 算是 sofa rpc 提供的一个特色功能，可以在**运行时**，给用户一个指定 provider 的机会，使用方式如下：

```java
RpcInvokeContext.getContext().setTargetURL("127.0.0.1:12201");
```

在运行时指定调用地址之后，sofa rpc 会优先使用改地址去发起调用。

```java
targetIP = (String) rpcInternalContext.getAttachment(".pinpoint");

if (isNotBlank(targetIP) {
    // 如果指定了调用地址
    providerInfo = selectPinpointProvider(targetIP, providerInfos);
    ClientTransport clientTransport = selectByProvider(message, providerInfo);
    if (clientTransport == null) {
        // 指定的不存在或已死，抛出异常
        throw unavailableProviderException(message.getTargetServiceUniqueName(), targetIP);
    }
    return providerInfo;
} 
```





#### 4.不断负载均衡，找出本次调用的 provider

一般情况下，到这一步时，已经通过 router 获取到一堆 provider 信息了（当然，如果配置了 pinpoint，代码也不会走到这里了），现在需要做的是，从这一堆 provider 中，挑选一个合适的节点来发起调用，即：通过负载均衡算法，选出一个节点来发起调用：

```java
protected ProviderInfo select(SofaRequest, List<ProviderInfo>){
    
    ...
    
    List<ProviderInfo> providerInfos = routerChain.route(message, null);

    ...
  
    
    do {
        // 通过负载均衡算法，得出最终要调用的节点
        providerInfo = loadBalancer.select(message, providerInfos);
        ClientTransport transport = selectByProvider(message, providerInfo);
        if (transport != null) {
            return providerInfo;
        }
        providerInfos.remove(providerInfo);
    } while (!providerInfos.isEmpty()); // 如果该节点不可用，则试试下一个
  
}

```

默认采用的是**权重随机**负载均衡算法（代码有删改），即每个 provider 都有一个调用权重（默认都是100），在做负载均衡时，随机挑选一个 provider，但是权重越大的 provider 被随机到的概率越高。

> 简述权重随机的算法实现：
>
> - 先 累加所有 provider 权重总和，记为值 w
> - 再 生成一个随机数 x，取值落在 0 到 w 之间
> - 随机数 x 落在哪个区间，即最终负载到该节点上
>
> 可以参考下图：abcde是 5 个 provider 节点，各个方块长度代表权重大小。然后将它们权重累加，最后 x 落在哪就是哪。
>
> ```ruby
> 0										x						   	w
> +---+------------+-----+------------+
> | a |      b     |  c  |      d     |
> +---+------------+-----+------------+
> ```









### 二、调用前经 filterChain 过滤

在选择完 provider 之后，要做的事情就是发起调用了，不过在调用之前，rpc 模块有一个经典的预处理操作扩展点 —— filterChain。

filterChain 不仅是 rpc 的经典模块，甚至可以说是任何网络交互的经典模块。**设计 filter 目的是在调用操作发生之前，加入一些定制化的预处理操作**，比如日志打印、鉴权、限流等等。

##### 链式传递数据

一般情况下，filterChain （责任链模式）的实现是一个链表，**链上每个处理器都可以对数据进行加工，然后传递给下一个处理器**。在 sofa  rpc 的实现中，使用 filterInvoker 来串起所有的 filter，可以类比 netty 的 HandlerContext 和 handler，同样也是责任链模式。

```java
@ThreadSafe
public class FilterInvoker implements Invoker {

    protected Filter                  nextFilter;
    protected FilterInvoker           invoker;
  
    ...
}
```

不过，这个地方的引用关系并非：

![image-20230505154553793](https://study-notes-picture.oss-cn-hangzhou.aliyuncs.com//notes_pics/image-20230505154553793.png)



而是这种方式：

![image-20230505154618009](https://study-notes-picture.oss-cn-hangzhou.aliyuncs.com//notes_pics/image-20230505154618009.png)



链式调用的代码如下，可以看到，调用 nextFilter 的 invoke 方法时，会把 nextInvoker 传递进去，这样就约束了 filter 中，务必使用 invoker.invoke 方法来实现链式调用：

```java
public SofaResponse invoke(SofaRequest request) throws SofaRpcException {
    if (nextFilter == null && invoker == null) {
        throw new SofaRpcException(RpcErrorType.SERVER_FILTER,
            LogCodes.getLog(LogCodes.ERROR_NEXT_FILTER_AND_INVOKER_NULL));
    }
    return nextFilter == null ?
        nextInvoker.invoke(request) :
        // 将 nextInvoker 传递进去
        nextFilter.invoke(nextInvoker, request);
}
```



##### 定制化决定是否加载

当然，不是每个 provider 或 consumer 都要加载所有的 filter，我们可以控制 filter 的 **needToLoad** 方法的返回值，来动态地决定该 filter 是否需要被加载到链中：

```java
public abstract class Filter {
    /**
     * Is this filter need load in this invoker
     * @param invoker Filter invoker contains ProviderConfig or ConsumerConfig.
     * @return is need load
     */
    public boolean needToLoad(FilterInvoker invoker) {
        return check();
    }
    ...
}
```

如果 filter 的 needtoLoad 结果为 false，则不会被加载：

```java
for (filter : filters) {
  
    if (filter.needToLoad(invokerChain)) {
        // 传递自身，完成整条链的构造
        invokerChain = new FilterInvoker(filter, invokerChain, config);
        // cache this for filter when async respond
        loadedFilters.add(filter);
    }
}
```





##### 最后一个Invoker

还有一个技术细节是，一般情况下，最后一个 filterInvoker 并不是普通的 invoker，而是一个专用的 invoker，该 invoker 就是用来**进行请求发送**（对于 provider 而言，则是请求执行）。 Consumer 端最后一个 filterInvoker 如下：

```java
public class ConsumerInvoker extends FilterInvoker {
  
    private ConsumerBootstrap consumerBootstrap;
    
    // 构造时传入 consumerBootstrap，而 consumerBootstrap 会持有 cluster
    public ConsumerInvoker(ConsumerBootstrap consumerBootstrap) {
        ...
        this.consumerBootstrap = consumerBootstrap;
    }

  
    @Override
    public SofaResponse invoke(SofaRequest sofaRequest) throws SofaRpcException {
        ...
        // 通过当前的 cluster 发出请求
        return consumerBootstrap.getCluster().sendMsg(providerInfo, sofaRequest);
    }
}
```

不过需要注意的是，该 invoker 发出请求，是委托 cluster 做的，而 cluster 则是把请求提交到 transport 层来发送的。







### 三、通过 transport 层发起调用

当整条 filter 链都处理完后，接下来就可以通过网络发送给对端了。在 sofa rpc 中，负责请求发送的是 transport 模块。**transport 本身是一个高层的抽象模块**，它其实并**不会关注底层**在通信时具体采用什么样的解决方案。它抽象出了底层通信的共性行为：

- 连接管理
- 网络调用

那么具体在进行网络传输时，我们可以选择的方案有很多，比如 dubbo3 的 triple，蚂蚁自己的 bolt，当然，<a href="https://github.com/sofastack/sofa-bolt">sofa-bolt</a> 是正统的、默认的底层通信模块。

我们可以看下 transport 具体发送请求的代码，这里以 bolt 方案的 sync 调用为例：

```java
protected SofaResponse doInvokeSync(SofaRequest request, InvokeContext invokeContext, int timeoutMillis)
    throws RemotingException, InterruptedException {
    return (SofaResponse) RPC_CLIENT.invokeSync(url, request, invokeContext, timeoutMillis);
}
```

上面真正发送请求的 RPC_CLIENT 对象，就是 bolt 框架提供的通信客户端对象。到这我们可以认为，**一个请求就已经从 biz 现成提交到 io 线程，并通过网卡发出去了，在等待一定的时间后，会拿到该请求的响应，并逐级返回给代理类**，我们 rpc 客户端就算结束自己的任务了。当然，里面还是有很多细节的，这里也只是看到了冰山的尖尖，下面内容非常非常非常非常多。

> 如果希望继续研究，可以把断点打到 bolt 客户端顶层抽象类 BaseRemoting 中。对于 sofa-bolt 的源码解析，我会放在另一个系列文章中，bolt 的源码也很精彩，很多设计也值得我们学习。







### X、总结

我们可以回顾下发起请求的流程，其实就干了三件事：

- 找出本次要调用的 provider 节点
- 调用前进行预处理
- 发起网络请求

![image-20230506113402977](https://study-notes-picture.oss-cn-hangzhou.aliyuncs.com//notes_pics/image-20230506113402977.png)

注意，本文重点说了请求的主线流程，忽略了很多细节，比如**请求的组装**、**provider tranport 的校验**、**rpc context 的设置和查阅**等，大家可以自行查阅源码。

