---
title: （一）Java 中 final 关键字的用法总结
date: 2023-01-12 00:07:59
tags:
    - java
    - jvm
    - 开发语言
categories: Java基础知识
---

<!--more-->

## 一、final 的基本用法
### 1、修饰类
final修饰一个类时，表示该类不能继承。比如下面的写法就是错误的：
```java
final class A{

}
class B extends A{

}
```
而且对于类、方法来说，abstract关键字和final关键字不能同时使用，因为矛盾。有抽象方法的abstract类被继承时，其中的方法必须被子类Override，而final不能被Override。
### 2、修饰方法
final 修饰的方法不能被覆盖或者重写，这也确保了父类中的方法被子类继承后发生异变（即重写）。也就是说，我们如果用 final 修饰某个方法，这个方法就是稳定的，不变的。

```java
class A{
    final public void printHelloWorld(){
        System.out.println("Hello,world!");
    }

}
class B extends A{
    @Override
    public void printHelloWorld(){
        System.out.println("Hello,moon!");
    }
}

```
上面这个子类的`printHelloWorld()`方法就会报错，因为父类中已经用 `final `修饰过了。但是想要重写也是可以的，如果父类中final修饰的方法同时访问控制权限为`private`，将会导致子类中不能直接继承到此方法，因此，此时可以在子类中定义相同的方法名和参数，此时不再产生重写与`final`的矛盾，而是在子类中重新定义了新的方法。比如：

```java
class A{
    private final void printHelloWorld(){
        System.out.println("Hello,world!");
    }

}
class B extends A{
    public void printHelloWorld(){
        System.out.println("Hello,moon!");
    }
}
```
这就是可以的，另外，类的`private`方法会隐式地被指定为`final`方法。因此，IDEA 会建议我们去掉关键字`final`，像下面这样：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/758cc23401414177a72b4125fa40a4af_1720253884434.png?token=ANB4BCLKMT7GSIIRKXDSW4DGRD67S)
### 3、修饰变量
`final`成员变量表示常量，只能被赋值一次，赋值后值不再改变。
当final修饰一个基本数据类型时，表示该基本数据类型的值一旦在初始化后便不能发生变化；如果final修饰一个引用类型时，则在对其初始化之后就不可以指向其它对象了，但该引用所指向的对象的内容是可以发生变化的。
final修饰一个成员变量（属性），必须要显示初始化。这里有两种初始化方式，一种是在变量声明的时候初始化；第二种方法是在声明变量的时候不赋初值，但是要在这个变量所在的类的所有的构造函数中对这个变量赋初值。比如：

```java
public class FinalTest01 {
    public static void main(String[] args) {
        final int a = 10;
        a = 11; //报错
        final B bb = new B(20);
        bb.b = 30;
        bb = new B(30); //报错
    }
}
class B {
    int b;
    public B(int b) {
        this.b = b;
    }
}
```
## 二、final 的优化问题
可以看一下这个程序：

```java
public class FinalTest01 {
    public static void main(String[] args) {
        String a = "hello-world";
        final String b = "hello-";
        String b1 = "hello-";
        String c = b+"world";
        String d = b1+"world";
        System.out.println(c == a);
        System.out.println(d == a);
    }
}
```
它的输出为：

> true
false

为什么第一个比较结果为true，而第二个比较结果为fasle？这里面就是`final`变量和普通变量的区别了，当`final`变量是基本数据类型以及`String类型`时，如果在编译期间能知道它的确切值，则编译器会把它当做编译期常量使用。也就是说在用到该`final`变量的地方，相当于直接访问的这个常量，不需要在运行时确定。
当然这里也同样涉及编译器对 String 类型的优化，不过这个不是这里的重点，不再赘述。

## 三、final 修饰参数
看下一段代码：
```java
public class FinalTest01 {
    public static void main(String[] args) {
        FinalTest01 finalTest01 = new FinalTest01();
        int a = 10;
        finalTest01.changePer(a);
        System.out.println(a);

    }
    public void changePer(final int i){
         i++;// 会报错,
        System.out.println(i);
    }
}
```
因为 changePer() 中的参数是 final 修饰过的，所以该参数在函数体内就无法被改变了。因为Java参数传递采用的是**值传递**，对于基本类型的变量，相当于直接将该变量的值进行了赋值。因此就算没有final修饰，在方法内部改变了变量i的值，也不会main 函数中的i。这在 Java 中是非常关键的点。值传递是 Java 的一大特性，这是 Java 设计的思想，即把程序安全交给了 Java 的设计师，而不是程序员。这也是和 C++非常大的一个区别。当然这也和本篇博客的主题没有太大关系，不再赘述。
## 四、final 修饰的变量和常量的关系
这个问题比较复杂，我会另开辟一篇文章专门讨论。

*全文完，感谢你的阅读。*

