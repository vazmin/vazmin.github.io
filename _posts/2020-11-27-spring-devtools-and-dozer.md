---
title: 分析spring devtools与dozer的兼容问题
date:   2020-11-27 10:45:55 +0800
categories:
  - java
tags:
  - spring-devtools classloader
---

某个项目开启spring devtools后，某些地方遍历列表抛出异常`ClassCastException`。

debug + google 知除了类型不一致，`ClassLoader`不一致也会抛`ClassCastException`。

发现当源对象`class A { List<B> b }`用dozer(`net.sf.dozer:dozer:5.5.1`)转换为目标对象`class C { List<B> b }`时，将转化后的对象的成员变量`b`执行遍历, 会抛出异常`ClassCastException`。

dozer转换后的对象使用的`ClassLoader`为`RestartClassLoader`,但成员变量`b`的是`Launcher$AppClassLoader`。

如果直接定义变量`b`,其`ClassLoader`为`RestartClassLoader`。明显dozer在转换对象创建成员变量`b`是使用了不同的`ClassLoader`。

阅读dozer的源码，知dozer通过反射封装成员变量的信息，然后匹配将成员变量的值赋值到目标对象。

记录字段的类名
```java
// since we are mapping some sort of collection now is a good time to decide
// if they provided hints
// if no hint is provided then we will use generics to determine the mapping type
if (fieldMap.getDestHintContainer() == null) {
  Class<?> genericType = fieldMap.getGenericType(BuilderUtil.unwrapDestClassFromBuilder(destObj));
  if (genericType != null) {
    HintContainer destHintContainer = new HintContainer();
    destHintContainer.setHintName(genericType.getName());
    FieldMap cloneFieldMap = (FieldMap) fieldMap.clone();
    cloneFieldMap.setDestHintContainer(destHintContainer); // should affect only this time as fieldMap is cloned
    fieldMap = cloneFieldMap;
  }
}
```

然后通过hintName加载类
```java
public static Class<?> loadClass(String name) {
  BeanContainer container = BeanContainer.getInstance();
  DozerClassLoader loader = container.getClassLoader();
  return loader.loadClass(name);
}
```

而`DozerClassLoader`通过`BeanContainer`的类加载器构造。
```java
public class BeanContainer {
  DozerClassLoader classLoader = 
      new DefaultClassLoader(getClass().getClassLoader());
}
```
然而spring devtools并没有包含 dozer的jar，即`BeanContainer`使用的是默认的类加载器`Launcher$AppClassLoader`。

显然只需要告诉devtools包含dozer的jar即可。

解决办法：
1. 在资源目录下创建`META-INF/spring-devtools.properties`
2. 将下面内容加到`META-INF/spring-devtools.properties`
```
restart.include.dozer=/dozer-[\\w\\d-\.]+\.jar
```

附：
 * [Customizing the Restart Classloader](https://docs.spring.io/spring-boot/docs/2.3.6.RELEASE/reference/html/using-spring-boot.html#using-boot-devtools-customizing-classload) 

























