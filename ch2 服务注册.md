

### 说明

- 对于 sofa rpc 的服务注册，同样存在两种开发模式：基于注解 or 基于xml文件
- sofaboot 对服务注册和引用整体的处理思路比较类似
  - 服务引用：
    - 对于 @SofaReference 服务引用：使用 BeanPP 手工创建并注入 proxy，创建 proxy 时会触发服务引用
    - 对于 xml 服务引用：通过 refParser 生成 ReferenceFactoryBean 的 BD。业务 bean 依赖注入 ReferenceFactoryBean，ReferenceFactoryBean 会在 afterPropertiesSet 阶段引入服务，返回代理

  - 服务注册：
    - 对于 @SofaService 服务注册：使用 BeanFactoryPP 手工放入 serviceFactoryBean，通过 spring 加载所有 bean 的阶段，serviceFactoryBean 会完成服务注册
    - 对于 xml 服务注册：通过 svcParser 生成 ReferenceFactoryBean 的 BD。业务 bean 依赖注入 ServiceFactoryBean。


而无论是 xml 还是注解，也都需要创建 sofa rpc 代理服务，并注入到业务 bean 中，实现服务的引用。两种开发模式，本质代理创建流程是一样的，区别在于**IOC 层面怎么触发 rpc 代理服务的创建**。









## 一、扫描项目中 rpc 服务注册

> 关键字：自动装配、beanPostProcessor、注解扫描、schema 扩展、BeadDefParser、注入 ServiceFactoryBean

### 目标一、扫描注解

对于注解方式注册项目中的 sofa rpc 服务，要发现这些服务，也是通过 beanPostProcessor 就可以发现这些 bean 是否使用了 @SofaService 注解。实际上 @SofaReference 服务引用的处理，其实本质就是这种方式实现的。

但是对于服务注册，sofaboot 并没有继续使用 **bean** PostPrcessor，而是使用了**beanfactory** PostProcessor。



##### 1.BeanFactoryPostProcessor 扫描所有 bean 元信息，检查 @SofaService

- 自动装配注入一个 BeanFactoryPostProcessor，其作用是于检测代码中的 @SofaService 注解

- **ServiceBeanFactoryPostProcessor** 会检测所有的 bean 元信息，检查是否有@SofaService，该过程发生在 refresh 中的 invokeBFPP 阶段：

  ```java
  public class ServiceBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
      @Override
      public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) throws {
          Arrays.stream(bf.getBeanDefinitionNames())
              .collect(Collectors.toMap(Function.identity(), bf::getBeanDefinition))
              .forEach((k, v) -> transformSofaBeanDefinition(k, v, bf));
      }
  }
  ```
  
- 扫描这些 bean 是否使用了 @SofaService，核心就是 **transformSofaBeanDefinition** 方法：

  ```java
  if (该 bean 来自 @Bean) {
      // 解析 @Bean 上有无 @SofaService
      generateSofaServiceDefinitionOnMethod(...);
  } else {
      // 解析普通 @Component 上有无 @SofaService
      generateSofaServiceDefinitionOnClass(...);
  }
  
  
  if (sofaServiceAnnotation == null) {
  ```

  

##### 2.发现 @SofaService 注解后，生成 ServiceFactoryBean，放入 spring 容器

ServiceBeanFactoryPostProcessor 经过扫描，发现有的 bean 使用了 @SofaService，则会收集**注解元数据（作为服务暴露的配置）**，并生成对应的 **ServiceFactoryBean** 的 BD，然后放入 spring 容器（BDMap 中）。

```java
// ServiceBeanFactoryPostProcessor#generateSofaServiceDefinition
if (sofaServiceAnnotation != null) {
    // 开始生成 bd
    builder = BeanDefinitionBuilder.genericBeanDefinition();
    builder.getRawBeanDefinition().setBeanClass(ServiceFactoryBean.class);
    // 放入 BDRegistry (lbf)
    lbf.registerBD(serviceId, builder.getBD())
}
```



##### 3.等 spring 加载所有 bean 阶段，会把该 ServiceFactoryBean 加载

spring 在 refresh 的尾声阶段，会加载 all 非懒加载的 bean，这个时候 ServiceFactoryBean 就会被 spring 加载和初始化，接下来的服务注册操作，就发生在 ServiceFactoryBean 的初始化阶段。





### 目标二、扫描 xml

##### 1.通过 spring scheam 扩展机制扫描 服务注册 xml 文件

xml方式的扫描依然靠的是 spring 的 schema 扩展机制，sofaboot 使用的是 `ServiceDefinitionParser` 来实现的。

> 严格来说，这里是模板模式，真正做这件事代码在基类 AbstractBeanDefinitionParser 中。这非常符合 spring 框架的设计思想，在 spring 源码中，海量使用了模板模式，像我们熟知的 reader 的 loadBeanDefinition 、resource 的 getInputStream 等等，这也是 OOP 多态思想的体现。

 在 loadBeadDefinition 阶段，spring 会把 classpath 下的 xml 配置文件拿来解析，这时候就包含了配置的 服务注册的xml 配置文件。



##### 2.扫描和解析 xml 文件，生成 serviceFactoryBean，放入 spring 容器

解析 xml 时会用到**自定义的 parser**，parser 会生成 serviceFactoryBean 类型的 beanDefinition，然后放入 spring 容器的BD注册器（BDMap）中：

```java
BeanDefinitionReaderUtils.registerBeanDefinition(definition, registry);
```



##### 3.等 spring 加载所有 bean 阶段，会把该 ServiceFactoryBean 加载

这一步和注解处理一样，都是等着 spring 的 `finishBeanFactoryInitialization` 阶段加载所有 bean，届时一定也会加载这个 ServiceFactoryBean，ServiceFactoryBean 在加载时会完成服务的注册。







## 二、服务注册

> 关键词：factoryBean、initializingBean、适配器模式、spring application listener、优雅启动、合适的注册时机

在上述第一个阶段，其实就完成了一件事了：发现代码中使用到了 sofa rpc 的服务注册（服务暴露），那就根据服务信息，往 spring 容器里放一个 ServiceFactoryBean，这就完事了。剩下的就静静等待 spring 加载这个 bean 就好了。

##### 1.加载 serviceFactoryBean，调用 afterPropertieSet 扩展点

服务注册的具体行为，也就发生在加载 factorybean 的 **afterpropertiesSet** （initializingBean 阶段）过程中：

```java
public class ServiceFactoryBean implements InitializingBean, FactoryBean, {

    @Override
    protected void afterPropertiesSet() {
        
        ComponentInfo ci = new ServiceComponent(...);
        // 发起服务注册
        sofaRuntimeContext.getComponentManager().register(ci);
    }
}
```



##### 2.在 afterPropertieSet 阶段通过 adapter 发起服务注册

在 afterPropertiesSet 过程中，ServiceComponent 会进行 activate 操作。在这里通过 adapter 进行服务的 outBinding，不过在此之前，还是要组装 providerConfig，以便等会要注册到注册中心上，这便是 adapter 的 **preOutBinding** 操作：

```java
public void preOutBinding(...) {

    ...
    // 组装 providerConfig
    ProviderConfig providerConfig = providerConfigHelper.getProviderConfig(...);
  
    try {
        // 将 providerConfig 缓存起来
        providerConfigContainer.addProviderConfig(uniqueName, providerConfig);
    } catch (Exception e) {
        ...
    }
}
```

preOutBinding 做完之后，就开始调用 **outBinding** 发起真正的服务注册：

```java
private void activateBinding() {
    ...

    for (Binding binding : bindings) {
      
        // 取出不同 rpc 框架的适配器
        BindingAdapter<Binding> bindingAdapter = this.bindingAdapterFactory
            .getBindingAdapter(binding.getBindingType());
      
        Boolean result;
        try {
            // 发起服务注册
            result = bindingAdapter.outBinding(service, binding, target,getContext());
        } catch (Throwable t) {
        }
      
      
        ...
    }
}
```



##### 3.adapter 调用 sofa rpc 发起服务注册，创建 rpc 服务端

adapter 在 outBinding 中发起服务注册，主要做了三件事：

- 生成服务唯一 id（如：me.yq.biz.facade.UserInfoService:1.0:BETA:bolt）
- 委托 rpc 模块进行服务暴露，创建 rpc 服务端
  - 构造 **providerInvoker**，该对象包含 provider 端 **filterChian**，是 rpc 请求的真正执行者
  - 构建 rpc server，包括组装**netty 服务端引导类**，创建 **biz 线程池** 等等
  - 将 providerInvoker 注册到 rpc server 中

- 委托 registry 客户端进行服务发布（**实际上并没有发布**）

```java
public Object outBinding(Object contract, RpcBinding binding, Object target,
                         SofaRuntimeContext sofaRuntimeContext) {

    // 创建服务唯一 id
    String uniqueName = providerConfigContainer.createUniqueName((Contract) contract, binding);
    ProviderConfig providerConfig = providerConfigContainer.getProviderConfig(uniqueName);
    processorContainer.processorProvider(providerConfig);

    try {
        // 委托 rpc 模块进行服务暴露，创建 rpc 服务端
        providerConfig.export();
    } catch (Exception e) {
    }

    List<RegistryConfig> registrys = providerConfig.getRegistry();
    for (RegistryConfig registryConfig : registrys) {
        Registry registry = RegistryFactory.getRegistry(registryConfig);
        registry.init();
        registry.start();
        // 发布到（多个）注册中心
        registry.register(providerConfig);
    }

    return Boolean.TRUE;
}
```



##### 4.在 spring 上下文刷新完毕后，进行服务注册

等到 spring 上下问加载完毕后，会发布 context refreshed event，这个时候 sofaboot 有专门的 spring 事件监听器对此进行监听，然后进行 2 步操作：

- 步骤1：启动 rpc server（默认 boltServer）
- 步骤2：将接口服务信息注册到 registry

```java
// com.alipay.sofa.rpc.boot.context.SofaBootRpcStartListener#onApplicationEvent
public void onApplicationEvent(SofaBootRpcStartEvent event) {
    ...
  
    //启动 rpc server
    serverConfigContainer.startServers();

    //设置允许服务发布
    providerConfigContainer.setAllowPublish(true);

    //服务发布
    providerConfigContainer.publishAllProviderConfig();
    
    ...
}
```

这里牵涉到服务的**优雅启动**，两个细节：

- rpc server 先启动，把 biz 和 io 线程池准备好，再去发布服务
- spring context 没加载完所有 bean，不应该让 consumer 的请求打进来。





##### 5.至此，整个 provider 启动和服务注册算是结束了
