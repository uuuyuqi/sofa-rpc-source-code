

### 说明

##### 1.网络传输数据的固定流程:star:

无论是什么 rpc 技术，或者说无论是什么通信技术，都离不开一个固定的数据收发套路：

```ruby
client: 业务层 --> 序列化 --> 编码 --> 发出请求
server: 业务层 <-- 反序列化 <-- 解码 <-- 收到请求
```

##### 2.本文重点在 sofa-rpc 部分

当一个 rpc 请求从客户端 transport 模块发出之后，经过网络传输，会提交到服务端。本文主要介绍<u>服务端拿到请求，执行请求，返回响应的流程</u>。对于一些通信层面的细节，比如 sofa-bolt 怎么收发数据，这里不做过多阐述。



##### 3.那么本文提到的 sofa-bolt 是什么

如果您不熟悉 netty，可以略过 netty、bolt 相关的内容，但是内容看起来可能有点跳跃。

sofa-bolt 是蚂蚁推出的开源通信框架，注意，其关注点在于通信流程，说白了主要负责发消息、收消息。而 sofa-bolt 本身基于 netty 构建的，而 netty 封装了底层的 IO 流程，**就用户来说，只需要编写一些 handler，来实现消息编解码、收发处理、连接管理、事件管理即可**。

当一条请求从 consumer 发出后，请求到 provider 所在机器的 12200 端口，那么 provider 应用就会从该端口拿到数据，这些数据提交到应用层之后，会先经过 codec 收集，**codec 每次收到数据后，先累加起来**，然后对其尝试解码。每次解码出来的数据，会顺着 pipeline 往下传，交给后续的入站处理器处理。而后续的**入站处理器（inBoundHandler），会把请求提交给真正的提供服务的业务 bean 来处理**，业务 bean 执行完，会回传响应，然后顺着 pipeline 出站处理器（outBoundHandler）来处理，最后经过 codec 编码，然后顺着网络回传到请求者了。

当然，这里也只是叙述大概流程流程，如果您使用过 netty，您肯定知道除此之外还有很多细节需要处理，比如 io 线程和业务线程的转换，同步异步调用的处理逻辑等等等等。



##### 4.io 线程、biz 线程是什么

io 和 biz 线程的区别在于线程的工作：

- io 线程就是用来做 io 操作的（这本文我们只讨论 java 语言层面的 io 线程哦~），如果您写过 nio 代码，您可以理解就是对监听到读写事件做处理的线程。而一般情况下，我们的 io 线程又会分为 boss、worker 线程，boss 会专注于连接建立，worker 则专注于已建立连接上产生的读写事件，大名鼎鼎的 reactor 模式通常就会基于 boss 和 worker 来实现。
- biz 线程就是用来业务处理的。比如 io 的 worker 线程收到一个业务请求，那么worker 线程就会把请求提交到 biz 线程池，然后 biz 线程来真正处理该请求。



##### 5.remoting 层是什么？command 对象是什么？

大多数情况下，remoting 层指的是框架的远程通信模块，sofa 也不例外。

而 command 对象是一个通信层对象，通常也就是通信协议的映射对象，他会持有序列化后的业务数据对象，并能够直接转为字节流。











## 一、服务端拿到请求

在请求分发给服务端业务侧之前，会做很多预处理操作。一般情况下，服务端 netty pipeline 中，处理请求的最后一个入站处理器，需要把请求提交给业务线程，业务线程来调用业务 bean 执行请求。这里我们直接从服务端将请求提交到 rpc 业务线程池后， rpc 业务线程（`SOFA-SEV-BOLT-BIZ-${port}-${index}`）开始执行的地方开始看起：

```java
// code in RpcRequestProcessor#doProcess
// netty 入站处理器检测到解码后的数据是 RpcRequestCommand 类型，
// 则会提交到 RpcRequestProcessor#doProcess 中执行
public void doProcess(final RemotingContext ctx, RpcRequestCommand cmd) throws Exception {
    
    ...
    
    // 1.反序列化 command 对象
    if (!deserializeRequestCommand(ctx, cmd, RpcDeserializeLevel.DESERIALIZE_ALL)) {
        // 失败则 return
        return;
    }
    
    // 2.分发给业务处理器
    dispatchToUserProcessor(ctx, cmd);
}
```

provider 业务线程拿到请求之后，总体处理分为 2 步：

- 反序列化通信对象（command），得到业务请求对象
- 将请求分发给业务处理器

> 当然，并不是所有请求都可以提交到业务线程池的。高并发场景下，**业务线程池会被打满**，而 sofa-bolt 将调用请求往 rpc 业务线程池提交时，会提交不进去，抛出一个RejectedExecutionException（**rpc 场景基本不会设置 taskQueue，而是用同步队列，线程打满就抛拒绝异常**），这个异常会被 bolt 的 io 线程 catch 掉，并直接给客户端返回一个 RpcResponseCommand 对象，并将该对象的返回状态码设置为SERVER_THREADPOOL_BUSY，在 sofabolt 客户端收到该 command 对象后，通过获取其中的返回码为 SERVER_THREADPOOL_BUSY，进而得知当前服务端 biz 线程池已经打满了。







### 1.反序列化

##### 1.业务线程进行反序列化

我们可以看到，**业务数据的反序列化操作，是 biz 线程执行**的，为什么不在 io 线程里面做掉，直接给业务线程一个即用的业务对象？

实际上，序列化、反序列化操作是有一定开销的，这个开销并非可忽略不计的，尤其对于一些大对象，或者低效的序列化器，这个时间占用还是不容小觑的。我们的 io 线程（说白了就是 worker 线程）是非常宝贵的，其执行效率直接影响了服务端性能，所以一般序列化反序列化操作都会在 biz 线程里面做。





##### 2.反序列化的过程是分级的、幂等的

首先需要说明一下，一个 rpc 请求数据包，会包含如下内容：

```ruby
- 控制字段
- class：请求对象全类名
- header：rpc 请求头
- content：请求对象内容
```

反序列化过程是分级进行的：sofa请求进来之后，会先反序列化 class（这里是一个 String，而非 Class 对象），目的是优先找到该请求对应的 processor（processor 这个概念我下面会说），然后再序列化 header 和 content。 源码中维护了一个 **序列化级别** 字段，用来表示本次需要序列化到什么地步：

```java
public void deserialize(long level) throws DeserializationException {
    if (level <= RpcDeserializeLevel.DESERIALIZE_CLAZZ) {
        this.deserializeClazz();
    } else if (level <= RpcDeserializeLevel.DESERIALIZE_HEADER) {
        this.deserializeClazz();
        this.deserializeHeader(this.getInvokeContext());
    } else if (level <= RpcDeserializeLevel.DESERIALIZE_ALL) {
        this.deserialize();
    }
}
```

反序列化级别如下：

If level <= RpcDeserializeLevel.DESERIALIZE_CLAZZ, only deserialize clazz - only one part.
If level <= RpcDeserializeLevel.DESERIALIZE_HEADER, deserialize clazz and header - two parts.
If level <= RpcDeserializeLevel.DESERIALIZE_ALL, deserialize clazz, header and content - all three parts.



而反序列化由于分级设计的存在，为了防止反复序列化导致的问题，故将所有级别的反序列化操作都做成了幂等的：

```java
public void deserializeClazz() throws DeserializationException {
    if (this.getClazz() != null && this.getRequestClass() == null) {
        try {
            // deserialize clazz
        } 
    }
}
public void deserializeHeader(InvokeContext invokeContext) throws DeserializationException {
    if (this.getHeader() != null && this.getRequestHeader() == null) {
        try {
            // deserialize header
        } 
    }
}
public void deserializeContent(InvokeContext invokeContext) throws DeserializationException {
    if (this.getRequestObject() == null) {
        try {
            // deserialize content
        }
    }
}
```





### 2.提交给业务处理器处理

反序列化后就直接提交给业务处理器，准备调用 rpc 服务端接口：

```java
dispatchToUserProcessor(ctx, cmd);
```

dispatchToUserProcessor 方法有两个参数，ctx 表示当前通信上下文，包含请求的 netty channel，这个将作为发送响应时的目的地；cmd 是已经反序列化后的通信对象。









## 二、服务端处理请求

提交给业务处理器后，后面要做两件事：

- 找出对应的业务处理器（这个在 bolt 中就是 UserProcessor）
- 业务处理器执行请求（实际上就是委托业务 bean 的方法来执行的）

```java
private void dispatchToUserProcessor(RemotingContext ctx, RpcRequestCommand cmd) {
    final int id = cmd.getId();
    final byte type = cmd.getType();
  
    // 1.获取业务处理器——BoltServerProcessor
    UserProcessor processor = ctx.getUserProcessor(cmd.getRequestClass());
    // rpc 处理器一般是 async 处理器
    if (processor instanceof AsyncUserProcessor) { 
        try {
            // 2.处理请求
            processor.handleRequest(
                processor.preHandleRequest(ctx, cmd.getRequestObject()),
                new RpcAsyncContext(ctx, cmd, this), 
                cmd.getRequestObject());
        } catch (...) {
            ...
        }
    }  
}
```

### 1.获取业务处理器 BoltServerProcessor

业务处理器由业务侧编写，然后注册到 sofa-bolt 中，专门用于处理 sofa-bolt 通信层收集来的请求。对于 sofa rpc 而言，rpc 服务端的业务处理器——**BoltServerProcessor**，其作用就是能根据请求签名，来找到对应的业务 bean 及其方法，来发起调用，并将结果进行返回。

> 业务处理器这一套流程，是 sofa-bolt 框架提供的一套使用规范。上层模块（sofa rpc）需要创建并依赖 RpcServer、RpcClient 对象，然后通过 sofa-bolt 提供的注册处理器方法，把业务处理注册进去，注册进去之后。这样，sofa-bolt 在通信层拿到数据后，就会提交给这个业务处理器。
>
> 把业务处理器往 sofa-bolt 里面注册的方法可以参考这段代码：
>
> ```java
> public void registerUserProcessor(UserProcessor<?> processor) {
>     UserProcessorRegisterHelper.registerUserProcessor(processor, this.userProcessors);
> }
> ```
>
> **这个处理器注册方法，也是连接了 sofa-rpc 和 sofa-bolt 的关键胶水方法**，这个地方注册了之后，sofa-bolt 在收到业务请求之后，就会提交到 sofa-rpc 的业务处理器中处理了，即业务 bean 就能拿到来自通讯层的请求了。
>
> 对这个地方感兴趣的可以参考 sofa-bolt 官网提供的<a href="https://www.sofastack.tech/projects/sofa-bolt/sofa-bolt-handbook/">用户手册</a>，总之 sofa-rpc，sofa-registry-client 都使用了这套方式来实现业务层和 sofa-bolt 对接的。

通常情况下，我们的 rpc 服务端，就会通过这种方式





### 2.BoltServerProcessor 处理请求（核心:exclamation::exclamation::exclamation:）

BoltServerProcessor 处理请求时，主要做了这些事情：

1. 记录当前正在处理器的请求数（方便**优雅停机**）
2. 检测当前服务器是否正处于关闭状态（告知节点停机，方便**客户端重试**）
3. 判断当前请求是否已经超时（如果**已经超时则丢弃**，无需再执行）
4. 根据请求唯一 id，查找 invoker（invoker 就是服务注册时生成的 **providerProxyInvoker，包含 filterChain**）
5. 从反射 cache 中取出即将调用的目标方法（该缓存在服务注册时生成）
6. 切换到目标方法的类加载器，发起服务端调用，得到调用结果（切换 classloader，个人觉得应该还是防止 rpc 调用模块，和业务 bean 模块不是一个 classloader，从而**可能抛出 classNotFound 或者 noSuchMoethod**）
7. 返回调用结果（在下一小节中会介绍）
8. 清理 context（**防止 threadlocal 导致内存泄露**）

代码如下，可读性非常高，大家可以直接看注释就好了：

```java
try { // 这个 try-finally 为了保证Context一定被清理
    processingCount.incrementAndGet(); // 统计值加1

    ...

    // 开始处理
    SofaResponse response = null; // 响应，用于返回
    Throwable throwable = null; // 异常，用于记录
    ProviderConfig providerConfig = null;
    String serviceName = request.getTargetServiceUniqueName();

    周
    try { // 这个try-catch 保证一定有Response
        invoke:
        {
            // 情况1：服务端已关闭
            if (!boltServer.isStarted()) { 
                throwable = new SofaRpcException(RpcErrorType.SERVER_CLOSED, ...);
                response = MessageBuilder.buildSofaErrorResponse(throwable.getMessage());
                break invoke;
            }
            // 情况2：请求已经超时
            if (bizCtx.isRequestTimeout()) { // 加上丢弃超时的请求的逻辑
                throwable = clientTimeoutWhenReceiveRequest(...);
                break invoke;
            }
            // 情况3：一切正常，可以调用
            //     情况3-1 找不到目标服务
            Invoker invoker = boltServer.findInvoker(serviceName);
            if (invoker == null) {
                throwable = cannotFoundService(appName, serviceName);
                response = MessageBuilder.buildSofaErrorResponse(throwable.getMessage());
                break invoke;
            }
            // 情况3-2 能找到目标服务
            if (invoker instanceof ProviderProxyInvoker) {
                providerConfig = ((ProviderProxyInvoker) invoker).getProviderConfig();
                // 找到服务后，打印服务的appName
                appName = providerConfig != null ? providerConfig.getAppName() : null;
            }
            // 情况3-2-1 找不到目标方法
            String methodName = request.getMethodName();
            Method serviceMethod = ReflectCache.getOverloadMethodCache(serviceName, methodName,
                request.getMethodArgSigs());
            if (serviceMethod == null) {
                throwable = cannotFoundServiceMethod(appName, methodName, serviceName);
                response = MessageBuilder.buildSofaErrorResponse(throwable.getMessage());
                break invoke;
            // 情况3-2-2 能找到目标方法，那么，直接开调用！
            } else {
                request.setMethod(serviceMethod);
            }

            // 真正调用
            response = doInvoke(serviceName, invoker, request);

            if (bizCtx.isRequestTimeout()) { // 加上丢弃超时的响应的逻辑
                throwable = clientTimeoutWhenSendResponse(...);
                break invoke;
            }
        }
    } catch (Exception e) {
        // 服务端异常，不管是啥异常
        LOGGER.errorWithApp(appName, "Server Processor Error!", e);
        throwable = e;
        response = MessageBuilder.buildSofaErrorResponse(e.getMessage());
    }

    // 将产生的 response 进行返回
    if (response != null) {
        RpcInvokeContext invokeContext = RpcInvokeContext.peekContext();
        isAsyncChain = CommonUtils.isTrue(invokeContext != null ?
            (Boolean) invokeContext.remove(RemotingConstants.INVOKE_CTX_IS_ASYNC_CHAIN) : null);
        // 如果是服务端异步代理模式，特殊处理，因为该模式是在业务代码自主异步返回的
        if (!isAsyncChain) {
            // 其它正常请求
            try { // 这个try-catch 保证一定要记录tracer
                asyncCtx.sendResponse(response);
            } finally {
                if (EventBus.isEnable(ServerSendEvent.class)) {
                    EventBus.post(new ServerSendEvent(request, response, throwable));
                }
            }
        }
    }
} catch (Throwable e) {
    // 可能有返回时的异常
    if (LOGGER.isErrorEnabled(appName)) {
        LOGGER.errorWithApp(appName, e.getMessage(), e);
    }
} finally {
    processingCount.decrementAndGet();
    ...
    RpcInvokeContext.removeContext();
    RpcInternalContext.removeAllContext();
}
```





### 3.处理完返回结果

在 BoltServerProcessor 处理请求的收尾阶段，会返回本次请求的响应对象。代码如下：

```java
public void sendResponse(Object responseObject) {
    if (alreadySend) {
        processor.sendResponseIfNecessary(
            this.ctx, 
            cmd.getType(), 
            processor.getCommandFactory().createResponse(responseObject, this.cmd)
        );
    }
    ...
}
```

我们关注核心的返回方法 `processor.sendResponseIfNecessary`，可以看到这个地方就是委托 netty 的 HandlerContext 来发送相应的，相比 channel.writerAndFlush，可以减少不必要的 handler 处理：

```java
public void sendResponseIfNecessary(final RemotingContext ctx, byte type,
                                    final RemotingCommand response) {
    final int id = response.getId();
    // 非 oneway （单向）请求才需要响应
    if (type != RpcCommandType.REQUEST_ONEWAY) {
        RemotingCommand serializedResponse = response;
        try {
            response.serialize();
        } catch (SerializationException e) {
            ...
        } catch (Throwable t) {
            ...
        }
        // 发送相应！
        ctx.writeAndFlush(serializedResponse).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                ...
                if (!future.isSuccess()) {
                    logger.error(...);
                }
            }
        });
    }
    ...
}
```

注意啦，这个地方的发送响应的 processor 就不是 BoltServerProcessor 了，这个是 sofa-bolt 的通用 requestProcessor。





### 4.至此，provider 响应请求就算结束了











