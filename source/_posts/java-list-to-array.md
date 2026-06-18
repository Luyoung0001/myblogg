---
title: "Java 基础：List 转 Array 的两种方式"
date: 2023-03-07 21:58:53
tags:
  - "Java"
  - "collection"
  - "array"
categories:
  - "Programming Languages"
---

<!--more-->

## 方法一：
List 提供了toArray的方法，可以使得ArrayList 转化为一个数组。

> Suppose x is a list known to contain only strings. The following code
> can be used to dump the list into a newly allocated array of String:

```java
 String[] y = x.toArray(new String[0]);

```
## 方法二：
List 还提供了类似的方法，也可以达到这个目的，但是这个方法比较繁琐：

```java
String[] y = (String[])x.toArray();
```
像上面这种写法，会报错：

> Exception in thread "main" java.lang.ClassCastException: [Ljava.lang.Object; cannot be cast to [Ljava.lang.String;

这是因为 Java 中的转型机制一次性只能转型一个对象，不能将一个集合转成另一个数组。因此想要转型成功，就得写循环，比如：

```java
public class Test01 {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("Hello");
        list.add("World");
        list.add("!");
        String[] strings = new String[list.size()];
        Object[] arr = list.toArray();
        for (int i = 0; i < list.size(); i++) {
            strings[i] = (String) arr[i];
        }
        for (String string : strings) {
            System.out.println(string);
        }
    }
}
```



