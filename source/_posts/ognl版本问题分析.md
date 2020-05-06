---
title: OGNL版本问题分析
date: 2019-05-28 10:48:04
categories: Java
tags:
 - 问题排查
---

## 问题描述：

在查询sql的时候遇到报错：

```sql
org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.builder.BuilderException: Error evaluating expression 'groupStatusList != null and groupStatusList.size() > 0'. Cause: org.apache.ibatis.ognl.MethodFailedException: Method "size" failed for object [0] [java.lang.IllegalAccessException: Class org.apache.ibatis.ognl.OgnlRuntime can not access a member of class com.google.common.primitives.Ints$IntArrayAsList with modifiers "public"] |
	at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:75) |
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:371) |
	at com.sun.proxy.$Proxy42.selectList(Unknown Source) |
	at org.mybatis.spring.SqlSessionTemplate.selectList(SqlSessionTemplate.java:198) |
	at org.apache.ibatis.binding.MapperMethod.executeForMany(MapperMethod.java:119) |
	at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:63) |
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:52) |
	at com.sun.proxy.$Proxy62.getGroupIdByUserIdAndGroupStatus(Unknown Source) |
	at com.yit.groupbuy.service.GroupBuyInnerServiceImpl.queryProcessingGroupsByUserId(GroupBuyInnerServiceImpl.java:606) |
	at com.alibaba.dubbo.common.bytecode.Wrapper22.invokeMethod(Wrapper22.java) |
 api undesigned exception.
```

<!-- more -->


对应的代码部分为：

```
groupBuyExtendMapper.
            getGroupIdByUserIdAndGroupStatus(userId, Ints.asList(GroupBuyState.PROCESSING.value()))
```



## 问题分析：

从规范及用法上来看，并没有什么问题，检查后发现在源码中：

```java
public static Object invokeMethod(Object target, Method method, Object[] argsArray)
    throws InvocationTargetException, IllegalAccessException
  {
    boolean wasAccessible = true;

    if (securityManager != null) {
      try {
        securityManager.checkPermission(getPermission(method));
      } catch (SecurityException ex) {
        throw new IllegalAccessException("Method [" + method + "] cannot be accessed.");
      }
    }
    if (((!Modifier.isPublic(method.getModifiers())) || (!Modifier.isPublic(method.getDeclaringClass().getModifiers()))) && 
      (!(wasAccessible = method.isAccessible()))) {
      method.setAccessible(true); // ---- (1)
    }

    Object result = method.invoke(target, argsArray); // ------(3)
    if (!wasAccessible) {
      method.setAccessible(false); // -----(2)
    }
    return result;
  }
```

method实际上是一个共享变量，也就是例子中的

public int java.util.Collections$SingletonList.size()方法

当第一个线程t1至(1)行代码允许method方法可以被调用，第二个线程t2执行至(2)将method的方法设置为不可以访问。接着t1又开始执行到(3)行的时候就会发生该异常。



## 解决方法：

mybatis3.2.x版本引用ognl的版本为2.6.9，Modifier.isPublic存在问题，ognl2.7版本已经修复,所以要将mybatis版本升级到mybatis3.3.x以上即可



## 参考文档：

[Mybatis OGNL导致的并发安全问题](https://zhuanlan.zhihu.com/p/30085658)



