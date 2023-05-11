

## 一、扫描项目里 rpc 的使用

> 关键字：自动装配、beanPostProcessor、注解扫描、schema 扩展、BeadDefParser、注入 referenceFactoryBean

### 目标一、扫描注解

对于 sofa rpc 注解方式的服务引用，一般使用 processor 来扫描代码，发现其中的服务引用。

##### 1.bean processor 后置扫描所有 bean 的注解

- 以自动装配的形式，往 spring 容器中注入一个 bean post processor

  - 扫描 @SofaReference 注解的使用，依靠的处理器是：`ReferenceAnnotationBeanPostProcessor`

    ```java
    // 检测和处理时机：BeforeInitialization（initializingBean 阶段）
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
          processSofaReference(bean);
          return bean;
      }
    ```

-  该 processor 可以对 ioc 容器里面所有的 bean 进行扫描检测

  - 检验过程借用了 spring 中 bean 字段的处理方法：`ReflectionUtils#doWithFields`

> 自动装配这块可以参考：sofa-boot-project/sofa-boot-autoconfigure/src/main/resources/META-INF/spring.factories，这里面比较核心的自动装配的类是：SofaRuntimeAutoConfiguration。



##### 2.检测是否存在 @SofaReference 注解

beanPostProcessor 做的事情，就是在 BeforeInitialization 阶段之前，检测这个 bean 是否存在 @SofaReference ，存在则说明你的代码里面使用了 sofa rpc 服务引用。



##### 3.存在则创建代理并注入

一旦 processor 发现一个 bean 存在 sofa rpc 的服务引用行为，则为该字段注入一个 **rpc 代理 bean**（核心）

- 是否存在服务引用，则可以校验该 bean 头顶是否有 @SofaReference 注解

  ```java
  if (field.isAnnotationPresent(SofaReference.class)) {
      return true;
  }
  ```

- 一旦存在，则**根据注解元数据（作为服务引用配置），创建该服务的代理**。创建代理的流程，是真正服务引用的流程，是 rpc 客户端启动的核心

  ```java
  Object proxy = createReferenceProxy(sofaReferenceAnnotation, interfaceType);
  ```

- 之后将服务代理，以反射的方式将代理注入到字段中

  ```java
  ReflectionUtils.setField(field, bean, proxy); // field 是一个 rpc 引用字段
  ```

  注意一个细节，创建代理时，内部会触发服务引用，而服务引用的统一入口代码如下：

  ```java
  ReferenceRegisterHelper.registerReference(reference,...)
  ```





### 目标二、扫描 xml

如果是 xml 方式引入，不像注解驱动那样，直接一个 @SofaReference 注解搞定，而是需要借助 spring 的 DI 机制，往某个 bean 中注入一个 rpc 服务代理 bean（具体是`ReferenceFactoryBean` ）。而这个 rpc 服务代理 bean，就是通过 xml 文件定义的。那么对于 xml 驱动的 spring 项目，就需要阅读项目内的 xml 文件中是否存在服务引用即可。

而这个 xml 文件是怎么处理的？这个服务代理 bean 又是怎么放入的？其实也是利用了 spring 的 **schema 扩展机制**，本身和自动装配差不多，也是 spi 那种形式往 spring 中注入自己的 xml 处理器。之后在 **beanDefReader 进行 beanDef load 操作的期间**，会把 classpath 下的 xml resource 进行扫描和读取，然后利用自定义的 xml 处理器来读取内容并**生成对应的 beanDef**，放入容器中。

> *schema 扩展机制：*
>
> 参考宋顺大佬的这篇博客 http://nobodyiam.com/2017/02/26/several-ways-to-extend-spring/



##### 1.扫描 xml 文件，发现服务引用，则注入一个 factoryBean 作为代理 bean 的 BeanDef

这部分是自定义 xml 处理器（`ReferenceDefinitionParser`）做的，解析完 xml 信息后，会把`ReferenceFactoryBean`的 BeanDef 放入 BeanDefMap 中。



##### 2.spring 加载所有 bean 阶段，会加载代理 factoryBean，加载时会创建代理

我们的业务 bean 会依赖 SofaReference 引用的服务 bean（ReferenceFactoryBean），这个服务 bean 本身是 factoryBean，并实现了 spring 的 InitializingBean 扩展点，会在 afterPropertiesSet 阶段，触发代理的创建：

```java
@Override
public void afterPropertiesSet() throws Exception {
    // 解析 binding param 参数
    Binding binding = bindingConverter.convert(element, bindingConverterContext);
    // 开启服务引用
    doAfterPropertiesSet()
}
```

核心操作就在 doAfterPropertiesSet() 方法中：

```java
@Override
protected void doAfterPropertiesSet() throws Exception {
    Reference reference = buildReference();
    
    ...

    reference.addBinding(bindings.get(0));
    proxy = ReferenceRegisterHelper.registerReference(reference, ...);
}
```

服务这部分代码和上一部分的注解驱动引用服务的代码如出一辙，底层共同依赖了`ReferenceRegisterHelper#registerReference`这个注册入口方法。









## 二、创建代理和 rpc 客户端

> 关键字：sofaboot 向 sofa rpc 过渡、sofaboot component、适配器模式、bootstrap、refer、类的主动使用、类加载、模块初始化、优雅停机钩子

##### 前置说明

- 我们知道，rpc 服务调用，本质就是调用本地的一个代理。rpc 客户端调用本地代理对象，然后本地代理再去发起网络通信，从而调用远程的服务端；服务端返回结果后，代理再把结果返回给业务层。因此，引用远程服务的秘密，就藏在代理类的创建流程中。
- 具体的流程仍然是 rpc 做的，sofaboot 提供了一些列胶水类，完成 sofaboot 向 rpc 模块的功能委托。我们可以粗略的认为：**sofaboot 要做的事情，是一些列前置动作：收集用户的 rpc 配置，然后将这些配置进行包装，然后提交给 sofa rpc 去做真正的服务引用**。
- 实际上 sofaboot 做了远不止这些，它单独维护了一套组件（Component）的生命周期，我们可以关注关键的步骤，也就是各个组件的 activate 方法。



##### 1.委托 adapter 发起服务引用，创建代理

sofaboot 到 sofa rpc  过渡（缝合）的地方是`BindingAdapter`类中进行的一系列 inBinding（引用服务）和 outBinding（暴露服务）的操作。我们可以看到，其实也就是一个**适配器模式**，将 a 系统和 b 系统缝合，实现功能的整合。

sofaboot 委托 adapter 缝合 sofa rpc，引用服务，并创建代理：

```java
private Object createProxy(Reference reference, Binding binding) {
    BindingAdapter<Binding> bindingAdapter = bindingAdapterFactory.getBindingAdapter(binding
        .getBindingType());

    Object proxy;
    try {
        // 通过适配器，委托 rpc 模块创建代理服务，并进行服务引用
        proxy = bindingAdapter.inBinding(reference, binding, sofaRuntimeContext);
    } finally {
    }
    return proxy;
}
```



##### 2.发起引用之前，组装 ConsumerConfig 作为服务引用的基础元信息：

BindingAdapter 是用来调用 sofa rpc 功能的适配器，委托 rpc 来创建真正的服务代理。**至此，sofaboot 基本完成了自己的使命，剩下的就是 sofa rpc 要做的事情**。

```JAVA
// in BindingAdapter
@Override
public Object inBinding(Object reference, RpcBinding binding,
                        SofaRuntimeContext sofaRuntimeContext) {

    // 1.组装 ConsumerConfig
    ConsumerConfigHelper helper = ac.getBean(ConsumerConfigHelper.class);
    ConsumerConfig consumerConfig = helper.getConsumerConfig(reference,binding);

    try {
        // 2.创建代理
        Object proxy = consumerConfig.refer();
        binding.setConsumerConfig(consumerConfig);
        return proxy;
    } catch (Exception e) {
        ...
    }
}
```



##### 3.组装 ConsumerConfig 触发 rpc 模块的初始化

在创建 ConsumerConfig 时，会加载父类`AbstractIdConfig`（ProviderConfig 同理），父类的 static 代码块里面调用了`RpcRuntimeContext.now();`，进而触发 RpcRuntimeContext 的类加载，而 RpcRuntimeContext 的 static 代码块中会对 **整个 rpc 模块做初始化**，著名的**优雅停机**也在此处进行。

为什么这里调用`.now`方法？因为 java 在机制上，约定**主动使用类才会触发类的加载**。这个地方，调用静态方法也算是主动使用，这样就可以触发类的加载，可能也算是一种**无奈之举**。。。

```java
// in AbstractIdConfig
static {
    RpcRuntimeContext.now();
}
```

我们重点看一下 RpcRuntimeContext 的 static 代码块，这块会在类加载尾声的类初始化阶段（clinit）执行：

```java
static {
    
    // 初始化一些上下文
    initContext();
  
    // 初始化其它模块
    ModuleFactory.installModules();
  
    // 增加jvm关闭事件（就是shutdownHook，注意 kill -9 时不会执行，-15才会执行）
    if (RpcConfigs.getOrDefaultValue(RpcOptions.JVM_SHUTDOWN_HOOK, true)) {
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override
            public void run() {
                ...
                destroy(false);
            }
        }, "SOFA-RPC-ShutdownHook"));
    }
}
```

sofa rpc 模块包含了 tracer、lookout等等，这些也都是 sofa stack 的能力。







##### 4.ConsumerConfig.refer() 发起服务引用并创建代理

刚刚中间第二部插了个 rpc 模块初始化，现在回到 ConsumerConfig.refer，这个行为是真正的服务引用！sofa rpc 在最开始，会使用一个引导类（启动器类） consumerBootstrap ，来引导 rpc 客户端发起服务引用。

```java
public T refer() {
    if (consumerBootstrap == null) {
        // Bootstraps 是一个工具类，会根据引用的 协议 类型，生成 consumerBootstrap
        // 这里可以参考
        consumerBootstrap = Bootstraps.from(this);
    }
    // 真正的服务引用！
    return consumerBootstrap.refer();
}
```





##### 5.refer过程中创建 cluster 和 proxyInvoker，并返回代理

###### （1）创建 cluster 和 proxyInvoker

我们做 rpc 调用，最核心的就是 cluster 类，比如 dubbo 的 FailoverCluster，sofa 也是如此。cluster 可以理解就是 provider cluster，主要用来对 provider 做了多节点的 cache，每次就是从 cluster 中选出provider 发起调用。

proxyInvoker 可以粗略的认为是 cluster 的装饰器对象，**最终生成的动态代理，调用的就是 proxyInvoker**（生成的动态代理，会持有 proxyInvoker 的引用，在调用服务时，便会委托 proxyInvoker 发起调用），然后去委托 cluster 发起调用。

```java
@Override
public T refer() {
    // DCL 防止并发创建
    synchronized (this) {
        ...

        try {
            // build cluster
            cluster = ClusterFactory.getCluster(this);
            // build listeners
            consumerConfig.setConfigListener(buildConfigListener(this));
            consumerConfig.setProviderInfoListener(buildProviderInfoListener(this));
            // init cluster （核心）
            cluster.init();
            // 构造Invoker对象（执行链）
            proxyInvoker = buildClientProxyInvoker(this);
            // 创建代理类（默认 javaassit 方式）
            proxyIns = (T) ProxyFactory.buildProxy(代理方式,接口,proxyInvoker);

            ...
        } catch (Exception e) {
            ...
        }
        if (consumerConfig.getOnAvailable() != null && cluster != null) {
            cluster.checkStateChange(false); // 状态变化通知监听器
        }
        RpcRuntimeContext.cacheConsumerConfig(this);
        return proxyIns;
    }
}
```

在创建 cluster 时，默认采用的是 failover cluster，支持重试的。重点我们要关注的是 cluster 的 init 步骤：

```java
public synchronized void init() {
    if (initialized) { // balking
        return;
    }
    // 构造Router链
    routerChain = RouterChain.buildConsumerChain(consumerBootstrap);
    // 负载均衡策略
    loadBalancer = LoadBalancerFactory.getLoadBalancer(consumerBootstrap);
    // 地址管理器
    addressHolder = AddressHolderFactory.getAddressHolder(consumerBootstrap);
    // 连接管理器
    connectionHolder = ConnectionHolderFactory.getConnectionHolder(consumerBootstrap);
    // 构造Filter链
    this.filterChain = FilterChain.buildConsumerChain(this.consumerConfig,
        new ConsumerInvoker(consumerBootstrap));

    ...

    // 启动重连线程
    connectionHolder.init();
    try {
        // 得到服务端列表
        List<ProviderGroup> all = consumerBootstrap.subscribe();
        if (CommonUtils.isNotEmpty(all)) {
            // 初始化服务端连接（建立长连接)
            updateAllProviders(all);
        }
    } catch (Exception e) {
        ...
    }


    // 如果check=true表示强依赖，也就是无可用提供者则直接报错，启动失败
    if (consumerConfig.isCheck() && !isAvailable()) {
        throw new SofaRpcRuntimeException(LogCodes.getLog(LogCodes.ERROR_CHECK_ALIVE_PROVIDER));
    }
}
```

这里需要解释下这些对象的用途：

- routerChain：路由约束对象，当发起 rpc 调用时，会经过 router 筛选，得出可用的 provider 列表
- loadblancer：负载均衡对象，经过 router 筛选出 provider 信息后，需要从这些 provider 中选出一个真正要去调用的 provider。默认是 auto 负载均衡，它来决策到底使用具体底层的那个负载均衡实现，一般都是权重随机
- addressHolder：服务端地址管理对象，其本身实现了监听器接口，可以在注册中心客户端中及时被回调，修改 provide 信息
- connectionHolder：连接管理对象，addressHolder 仅仅是对地址管理，拿到的其实就是个 url，真正通信层连接管理还是 connectionHolder 来做的
- filterChain：过滤器链对象，过滤器可以在 rpc 调用的前后插入定制化操作，是服务治理的核心实现方式之一
- 重连线程：connectionHolder 会在后台为当前服务启动一个 RC 线程，专门检测连接的状态



###### （2）创建代理并返回

我们可以看到，在 refer 的最后一步，创建代理并进行了返回：

```java
@Override
public T refer() {

    ...

    try {

        cluster = ClusterFactory.getCluster(this);
				...
        proxyInvoker = buildClientProxyInvoker(this);
        // 创建代理类（默认 javaassit 方式）
        proxyIns = (T) ProxyFactory.buildProxy(代理方式,接口,proxyInvoker);

    }

    ...
    
    return proxyIns;

}
```

创建代理的核心，就是把 proxyInvoker 传进去，作为代理对象的调用入口。用户发起 rpc 调用时，底层调用的就是这个地方传入进去 proxyInvoker。在这里，dubbo 的处理也差不多，大家的玩法都大差不差，dubbo 源码如下：

```java
private transient volatile Invoker<?> invoker;

private T createProxy(Map<String, String> map) {
    ...
    invoker = cluster.join(...); // cluster#join 可以理解是在收集 router 信息
    ...
    return (T) proxyFactory.getProxy(invoker); // 创建代理，并传入 invoker
}

```











##### 6.完结

动态代理创建完毕，就持有了 cluster（proxyInvoker），至此，一个代理对象就算创建完了，而这个代理对象，可以认为就是狭义上的 rpc 客户端。创建完毕后，然后调用栈一级一级返回，最终回到 sofaboot，给一个消费者 bean，注入了这个代理。至此，完成了服务的引用。

不过说到现在，我所说的“服务的引用”，是从开发者角度，他们的业务消费者 bean 中，引入 sofa rpc 提供者，确切来说，只是完成了代理 bean 的注入，或者从更高维度说，目前做的事情是**打通了业务代码和框架代码**。但是还没有真正执行 rpc 层面的服务引用，请继续往下看。







##### 补充：代理类的大致内容 :laughing::laughing::laughing:

生成的代理类大概如下：

```java
// 假设代理的 provider 方法是 Test#test
public class Test_proxy_0 extends Proxy implements Test {
    public Invoker proxyInvoker = 上一步生成的 proxyInvoker;
    private Method method = ReflectUtils.getMethod(TestService.class, "test", new Class[]{String.class});

    
    // 被代理的方法，参数类型是 XXX
    public String test(XXX arg) {
        Class<?> clazz = TestService.class;
        Method methodName = this.method;
        Class<?>[] argTypes = new Class[1];
        Object[] args = new Object[]{arg};
        argTypes[0] = XXX.class;
        SofaRequest request = MessageBuilder.buildSofaRequest(clazz, methodName, argTypes, args);
        SofaResponse response = this.proxyInvoker.invoke(request);
        if (response.isError()) {
            throw new SofaRpcException(199, response.getErrorMsg());
        } else {
            Object appResponse = response.getAppResponse();
            if (appResponse instanceof Throwable) {
                throw (Throwable)var8;
            } else {
                return (String)var8;
            }
        }
    }
}
```

其代理步骤如下：

```java
SofaRequest request = MessageBuilder.buildSofaRequest(clazz, methodName, argTypes, args);
```

这个过程其实就是不断委托的过程，最终由 cluster 提交给 IO 线程池进行发送，然后默认会阻塞等待 IO 线程池的相应结果。









## 三、服务引用（订阅）和更新

> 关键字：引导类.subscribe()、创建注册中心客户端、发起服务注册

现在我们已经知道，sofa rpc 服务引用的大致流程是：

```ruby
sofaboot 扫描并触发
===> 通过适配器类，组装 consumerConfig
===> 发起服务 refer
===> 触发 rpc 模块的加载
===> 建立 cluster
===> 完成代理创建，返回给业务 bean
```

其中建立和初始化 cluster 时，会做一系列重要操作。这里，就包含了服务引用。在建立 cluster 的流程中，角落里有一串代码，蕴含着巨大能量：

```java
// com.alipay.sofa.rpc.client.AbstractCluster#init

// 启动重连线程
...
try {
    
    List<ProviderGroup> all = consumerBootstrap.subscribe();
    if (all 非空) {
        updateAllProviders(all);
    }
} catch (...) {
    ...
} catch (...) {
    ...
}
```

我们的服务发现，便是从这个地方开始。



##### 说明：服务订阅的整体设计思路

整个服务发现和更新过程，其实是 sofa rpc 委托  **注册中心客户端registry client** 来做的。这里就引入了 sofastack 的另一位成员，**sofa registry client**，其作为 registry 项目下的子项目，负责和 registry 做交互。sofa rpc 也利用其来 sofa registry （本文我就简称 registry 啦）通信：

具体的订阅思路大概得设计思路是：

- sofa rpc 的 subscriber（cluster 中的 addressHolder）需要监听 接口 的 providers 信息

- sofa rpc 先往 registry client 中注册一个 callback，这个 callback 只做一件事：通知 subscriber

- registry client 定期和 registry 通信，监听目标数据

  > 监听的方式，就是起一个后台线程，不停歇地和 registry 交互。通信层面可能是 http 长轮询或者 rpc，反正是某种推拉机制来实现监听。不过 sofa registry client 采用是 rpc，也是依赖了 bolt 这套通信框架。

- registry client 拿到最新数据，回调 callback，callback 会通知 rpc 的 subscriber（即 cluster）

这也算业界比较通用的设计方式了。



##### 1.引导类发起服务订阅

初始化 cluster 的时候，会进行服务订阅，也就是`consumerBootstrap.subscribe();`这句代码中，开始从注册中心获取 provider 信息，真正的开始了服务引用（服务发现）！

从订阅的源码中可以看到，先获取 registry 的信息，然后开始服务订阅。

```JAVA
public List<ProviderGroup> subscribe() {
    ...
    
    List<RegistryConfig> registryConfigs = consumerConfig.getRegistry();
    if (CommonUtils.isNotEmpty(registryConfigs)) {
        // 从多个 registry 发起服务的订阅
        result = subscribeFromRegistries();
    }

    return result;
}
```



##### 2.订阅前确定要订阅的信息

rpc 客户端在向注册中心发起服务订阅的时候，会先整合一下要订阅的信息，其实就是**接口唯一id**，大致如下：

```bash
# 规定接口唯一 id 是：
接口全类名:版本[:uniqueId]@协议

# 比如默认 bolt 协议的接口 id：
me.yq.common.api.CommonService:2.0:AS_UIS_USERINFO_QRY@DEFAULT
```



##### 3.生成 callback 来通知自己

简单点来理解就是，sofa rpc 告诉 registry 客户端：你帮我盯着这个数据 x，一旦你发现这个 x 变化了，你就赶紧回调我给你的这个 callback，你调了它之后，我自己的 cluster 就能拿到最新数据了！说到底也就是一套回调机制，如果 registry client 发现服务或者监听到服务变动，就会调用这个回调。

sofa rpc 生成 callback 的源码如下，callback 中包含了自己的 cluster：

```JAVA
// see  com.alipay.sofa.rpc.registry.sofa.SofaRegistry#subscribe
callback = new SofaRegistrySubscribeCallback();
callback.addProviderInfoListener(serviceName, config, providerInfoListener);
```



##### 4.往 registry 客户端塞入 callback

callback 准备好之后，就要塞到 registry client 中了，只不过在塞入时，是按照 k-v 的形式塞入的，k 则是上层（sofa rpc）需要监听的接口，v 则是发生变动后的接口回调。k-v 会包装成一个 `SubscriberRegistration` 对象。

```java
// see  com.alipay.sofa.rpc.registry.sofa.SofaRegistry#subscribe
SubscriberRegistration subscriberRegistration = new SubscriberRegistration(serviceName, callback);
...
listSubscriber = SofaRegistryClient.getRegistryClient(...).register(subscriberRegistration);
```



##### 5.异步获取服务 provider 信息

依靠 registry client 来异步刷新 provider 信息，完成服务信息的装入。

> 说是一部获取 provider 信息，实际上源码里还是用 countDownlacth 等了一下，这块就没再继续往下研究了~





##### 6.异步更新 provider 信息

一旦监听到 provider 信息变动（即便是首次监听），就会更新本地 provider 信息。主要做更新的有两个对象：

- addressHolder 修改地址列表
- connectionHolder 新增或删除链接（删除连接时优雅删除的）

```java
public abstract class AbstractCluster extends Cluster {
    @Override
    public void updateProviders(ProviderGroup providerGroup) {
        checkProviderInfo(providerGroup);
        
        if (ProviderHelper.isEmpty(providerGroup)) {
            addressHolder.updateProviders(providerGroup);
            closeTransports();
        } else {
            addressHolder.updateProviders(providerGroup);
            connectionHolder.updateProviders(providerGroup);
        }
        ...
    }
}
```



##### 7.完结

至此，服务引用阶段算是结束了。



