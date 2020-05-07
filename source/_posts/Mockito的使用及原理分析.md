---
title: Mockito的使用及原理分析
date: 2017-10-01 14:33:17
categories: 单元测试
tags:
 - 单元测试 
 - Mockito
---

本文包含以下内容：

* 如何使用Mockito写单元测试
* Mockito实现原理浅析
* 模仿Mockito实现mock功能

<!-- more -->

## 前言
上一篇讲述了如何编写健壮的单元测试，其中解决外部数据依赖的方式就是Mock数据返回，那么具体如何Mock数据呢？其实现机制又是怎样的呢？

Mock最常用的框架之一是Mockito，以下分析都将基于Mockito展开。

## Mockito

一次完整的Mock，包括

* 设定目标
* 设置消费条件
* 预期返回结果
* 消费并检验返回结果

我们首先看一个最简单的的例子来看下如何使用Mockito来进行Mock数据返回：

```Java
    when(productService.getProductInfo(any())).thenAnswer(invocationOnMock -> {
                List<ProductInfo> productInfos = new ArrayList<>();
                return productInfos;
            });
    
    //...        
    List<ProductInfo> productInfos  = productService.getProductInfo(1);
```

    
在上述例子中，这些条件是如何对应的呢？

* 设定目标 > `List<ProductInfo> productInfos`
* 设置消费条件 -> `productService.getProductInfo(any())`
* 预期返回结果 -> `thenAnswer(...)`
* 消费并检验返回结果 -> `productService.getProductInfo(1)`

所以不难发现，Mockito主要就是通过Stub打桩，通过方法名加参数来准确的定位测试桩然后返回预期的值

## 提出疑问

我们都知道，Java的程序调用是用堆栈来实现的，那么是不是有这样的疑问：
> 
`when()`消费的应该是`productService.getProductInfo()`函数的返回值，对其内部实现并不感知的，那么它是如何来准确通过函数名加参数的条件来打桩的呢？

## Mockito的实现原理

通过查看代码，我们不难发现`mock`的入口是在`MockitoCore.java`中的这段代码：

```Java
public <T> T mock(Class<T> typeToMock, MockSettings settings) {
        if (!MockSettingsImpl.class.isInstance(settings)) {
            throw new IllegalArgumentException("Unexpected implementation of '" + settings.getClass().getCanonicalName() + "'\n" + "At the moment, you cannot provide your own implementations of that class.");
        }
        MockSettingsImpl impl = MockSettingsImpl.class.cast(settings);
        MockCreationSettings<T> creationSettings = impl.confirm(typeToMock);
        T mock = createMock(creationSettings);
        mockingProgress().mockingStarted(mock, creationSettings);
        return mock;
    }
```

在这里mock函数接收两个参数typeToMock（mock类型）和settings（设置内容），目标是创建一个mock对象。跟随第六行`createMock()`进入`MockUtil.java`中，可以看见动态调用了`org.mockito.internal.creation.cglib`类：

```
public <T> T createMock(MockCreationSettings<T> settings, MockHandler handler) {
        InternalMockHandler mockitoHandler = cast(handler);
        new AcrossJVMSerializationFeature().enableSerializationAcrossJVM(settings);
        return new ClassImposterizer(new InstantiatorProvider().getInstantiator(settings)).imposterise(
                new MethodInterceptorFilter(mockitoHandler, settings), settings.getTypeToMock(), settings.getExtraInterfaces());
    }
```
上述代码中根据MockHandler创建InternalMockHandler后，通过cglib代理来执行mock的流程，而在`imposterise`实现中调用了`createProxyClass()`的方法，而`createProxyClass()`方法中的是实现是创建Mock对象的关键所在了：

```
public <T> T imposterise(final MethodInterceptor interceptor, Class<T> mockedType, Class<?>... ancillaryTypes) {
        Class<Factory> proxyClass = null;
        Object proxyInstance = null;
        try {
            setConstructorsAccessible(mockedType, true);
            // 创建代理对象
            proxyClass = createProxyClass(mockedType, ancillaryTypes);
            proxyInstance = createProxy(proxyClass, interceptor);
            return mockedType.cast(proxyInstance);
        } catch (ClassCastException cce) {
            // ...此处省略
        }
    }
```

点进去`createProxyClass()`查看具体的代码实现块，可以发现这里进行了很多的`enhancer`设置。Enhancer允许为非接口类型创建一个Java代理。Enhancer动态创建了给定类型的子类但是拦截了所有的方法。

```
public Class<Factory> createProxyClass(Class<?> mockedType, Class<?>... interfaces) {
        if (mockedType == Object.class) {
            mockedType = ClassWithSuperclassToWorkAroundCglibBug.class;
        }
        
        Enhancer enhancer = new Enhancer() {
            @Override
            @SuppressWarnings("unchecked")
            protected void filterConstructors(Class sc, List constructors) {
                // Don't filter
            }
        };
        Class<?>[] allMockedTypes = prepend(mockedType, interfaces);
		enhancer.setClassLoader(SearchingClassLoader.combineLoadersOf(allMockedTypes));
        enhancer.setUseFactory(true);
        if (mockedType.isInterface()) {
            enhancer.setSuperclass(Object.class);
            enhancer.setInterfaces(allMockedTypes);
        } else {
            enhancer.setSuperclass(mockedType);
            enhancer.setInterfaces(interfaces);
        }
        enhancer.setCallbackTypes(new Class[]{MethodInterceptor.class, NoOp.class});
        enhancer.setCallbackFilter(IGNORE_BRIDGE_METHODS);
        if (mockedType.getSigners() != null) {
            enhancer.setNamingPolicy(NAMING_POLICY_THAT_ALLOWS_IMPOSTERISATION_OF_CLASSES_IN_SIGNED_PACKAGES);
        } else {
            enhancer.setNamingPolicy(MockitoNamingPolicy.INSTANCE);
        }

        enhancer.setSerialVersionUID(42L);
        // ... 省略代码
    }
```
而通过enhancer是如何创建Mock类对象的呢？

```
if (gen == null) {  
    byte[] b = strategy.generate(this);  
    String className = ClassNameReader.getClassName(new ClassReader(b));  
    getClassNameCache(loader).add(className);  
    gen = ReflectUtils.defineClass(className, b, loader);  
    }
```
在上述代码中，使用`strategy.generate(this)`生成字节码的字节数组，再通过`ReflectUtils.defineClass(className, b, loader)`将字节数组转换成class对象，以此来创建Mock类。

所以，Mock本质上是一个Proxy代理模式的应用。

Proxy模式，是在对象提供一个proxy对象，所有对真实对象的调用，都先经过proxy对象，然后由proxy对象根据情况，决定相应的处理，它可以直接做一个自己的处理，也可以再调用真实对象对应的方法。

所以Mockito本质上就是在代理对象调用方法前，用stub的方式设置其返回值，然后在真实调用时，用代理对象返回起预设的返回值。

### 验证

查看`when()`源码

```Java
public <T> OngoingStubbing<T> when(T methodCall) {
        MockingProgress mockingProgress = mockingProgress();
        mockingProgress.stubbingStarted();
        @SuppressWarnings("unchecked")
        OngoingStubbing<T> stubbing = (OngoingStubbing<T>) mockingProgress.pullOngoingStubbing();
        if (stubbing == null) {
            mockingProgress.reset();
            throw missingMethodInvocation();
        }
        return stubbing;
    }
```
发现所有的methodCall或被转换成OngoingStubbing对象，而OngoingStubbing存储了哪些信息呢？

```Java
public Object handle(Invocation invocation) throws Throwable {
     if (invocationContainerImpl.hasAnswersForStubbing()) {
         ...
    }
     ...
     InvocationMatcher invocationMatcher = matchersBinder.bindMatchers(
            mockingProgress.getArgumentMatcherStorage(),
           invocation
    );
   mockingProgress.validateState();
    // if verificationMode is not null then someone is doing verify()
   if (verificationMode != null) {
   ...
      }
    // prepare invocation for stubbing   invocationContainerImpl.setInvocationForPotentialStubbing(invocationMatcher);
  OngoingStubbingImpl<T> ongoingStubbing = 
  new OngoingStubbingImpl<T>(invocationContainerImpl);
mockingProgress.reportOngoingStubbing(ongoingStubbing);
   ...
}
```
查看上面这段代码，可以发现方法调用的信息(invocation)对象被用来构造invocationMatcher对象，最终传递给了ongoingStubbing对象。完成了stub信息的保存。

所以Mockito在构造时，不仅仅保存了方法的返回值，还做了大量处理，保存了stub的调用信息，才能准确定位。

而查看thenAnswer的代码，发现了这样的Demo:

```Java
    public Integer answer(InvocationOnMock invocation) throws Throwable {
        return (Integer) invocation.getArguments()[0];
    }
```

那么我们就应该可以在answer中拿到invocation的信息啊，于是我尝试了一下：

```Java
    @Test
    public void test2() {
        LinkedList mock = mock(LinkedList.class);
        when(mock.add(anyString())).thenAnswer(new Answer() {
            public Object answer(InvocationOnMock invocation) {
                Object[] args = invocation.getArguments();
                Object mock = invocation.getMock();
                return "called with arguments: " + args;
            }
        });

        System.out.println(mock.add("foo"));
    }
```
发现确实可以在方法中拿到调用函数以及参数信息。

所以在Mockiton中，在设置条件时，Mockito并不是对内部实现不感知，相反，保存了参数名以及入参信息，最终来构建stub，返回信息。

## 自己动手写Mockito Demo

在学习了Mockito实现原理之后，发现其实它本质上就是通过代理 + 反模式打桩实现的。那么可以自己实现一个Mockito么？

参考相关资料后，发现应该是可行的，并找到类似材料，那么试试吧。

### 实现 mock
Mock的实现关键是，实现动态代理，被 mock 的对象只是“假装”调用了该方法，然后返回假的值。

可以使用cglib来进行动态代理。通过class对象创建该对象的动态代理对象，然后设置该对象的父类与回调即可。并在回调函数中定义拦截器，实现自定义逻辑。

```Java
public class Mockito {

    /**
     * 根据class对象创建该对象的代理对象 
     * 1、设置父类；2、设置回调 
     * 本质：动态创建了一个class对象的子类 
     *
     * @return
     */
    public static <T> T mock(Class<T> clazz) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(new MockInterceptor());
        return (T)enhancer.create();
    }

    private static class MockInterceptor implements MethodInterceptor {
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            return null;
        }
    }
}
```

### 实现 stub

首先定义一个类，来表示对函数的调用，重写equals()方法，通过函数名 + 参数列表来判断调用是否相同。

```Java
public class Invocation {
    private final Object mock;
    private final Method method;
    private final Object[] arguments;
    private final MethodProxy proxy;

    public Invocation(Object mock, Method method, Object[] args, MethodProxy proxy) {
        this.mock = mock;
        this.method = method;
        this.arguments = copyArgs(args);
        this.proxy = proxy;
    }

    private Object[] copyArgs(Object[] args) {
        Object[] newArgs = new Object[args.length];
        System.arraycopy(args, 0, newArgs, 0, args.length);
        return newArgs;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == null || !obj.getClass().equals(this.getClass())) { return false; }
        Invocation other = (Invocation)obj;
        return this.method.equals(other.method) && this.proxy.equals((other).proxy)
            && Arrays.deepEquals(arguments, other.arguments);
    }

    @Override
    public int hashCode() {
        return 1;
    }
}
```

接下来，在 MockInterceptor 类中，需要做两个操作。

1. 为了设置方法的返回值，需要存放对方法的引用（Invocation）
2. 调用方法时，检查是否已经设置了该方法的返回值（results）。如果设置了，则返回该值。

```Java
public class Mockito {

    private static Map<Invocation, Object> results = new HashMap<Invocation, Object>();
    private static Invocation lastInvocation;

    public static <T> T mock(Class<T> clazz) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(new MockInterceptor());
        return (T)enhancer.create();
    }

    private static class MockInterceptor implements MethodInterceptor {
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            Invocation invocation = new Invocation(proxy, method, args, proxy);
            lastInvocation = invocation;
            if (results.containsKey(invocation)) {
                return results.get(invocation);
            }
            return null;
        }
    }

    public static <T> When<T> when(T o) {
        return new When<T>();
    }

    public static class When<T> {
        public void thenReturn(T retObj) {
            results.put(lastInvocation, retObj);
        }
    }
}
```

### 测试

测试用例如下：

```Java
    @Test
    public void test() {
        Calculate calculate = mock(Calculate.class);
        when(calculate.add(1, 1)).thenReturn(1);
        Assert.assertEquals(1, calculate.add(1, 1));
    }
```

## 其他

Mockito打桩返回的方式有很多，这边主要关注了经常使用的`thenAnswer()`函数，至于其他`thenReturn()`、`thenThrow()`、`thenCallRealMethod()`、`then()`函数，基本类似。


