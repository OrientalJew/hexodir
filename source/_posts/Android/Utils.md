---
title: Utils
date: 2018-08-03
tags:
- 工具类
categories:
- Android
---

<!-- toc -->

#### 校验重复方法签名

该方法取自EventBus3.0，原理是通过HashMap的put方法，在相同key的情况下，会返回旧的数据；

```
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
    // 构造方法签名
    methodKeyBuilder.setLength(0);
    methodKeyBuilder.append(method.getName());
    methodKeyBuilder.append('>').append(eventType.getName());

    String methodKey = methodKeyBuilder.toString();
    Class<?> methodClass = method.getDeclaringClass();
    // 放进HashMap查看是否有重复
    Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
    // 如果方法签名重复，并且两个方法是在同一个类或者存在父子或接口继承关系
    if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
        // Only add if not already found in a sub class
        return true;
    } else {
        // Revert the put, old class is further down the class hierarchy
        subscriberClassByMethodKey.put(methodKey, methodClassOld);
        return false;
    }
}
```
