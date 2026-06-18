---
title: "Java 基础（三）：抽象类与构造方法"
date: 2023-01-13 17:35:26
tags:
  - "Java"
  - "abstract class"
  - "OOP"
categories:
  - "Programming Languages"
---

<!--more-->

## 〇、前言
在Java中，可以通过两种形式来体现OOP的抽象：**接口**和**抽象类**。这两类概念既有很多的相似处又有很多的不同，下面用具体的例子来说明这两者的区别和联系。
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/4e87bf9914bf471292df6d97d80e5842_1720253850025.png?token=ANB4BCOB3OSYUMKZHCAF3YDGRD65Y)
## 一、抽象类
抽象类可以理解为一种**半抽象**，就是说它不是完全抽象的。它的里面可以含有完整定义的方法。比如：

```java
public class Test01 {
    public static void main(String[] args) {
        B b = new B();
        System.out.println(b.a);
        b.hello();
    }
}
abstract class A{
    public int a = 10;
    public void hello() {
        System.out.println("hello!");
    }
}
class B extends A{

}

```
当然，如果只能定义完整的方法，它身为抽象类就失去了意义。当抽象类中包含抽象方法时：
如果继承抽象类的是一个非抽象类，那么子类中必须给出实现：

```java
public class Test01 {
    public static void main(String[] args) {
        B b = new B();
        System.out.println(b.a);
        b.hello();
        b.move();
    }
}
abstract class A{
    public int a = 10;
    public void hello() {
        System.out.println("hello!");
    }
    abstract public void move();
}
class B extends A{
// 必须得实现 move 方法；
    @Override
    public void move() {
        System.out.println("Move!");
    }
}
```
如果继承抽象类的依然是个抽象类，那么就没必要实现了：

```java
public class Test01 {
    public static void main(String[] args) {
        B b = new B();
        System.out.println(b.a);
        b.hello();
        b.move();
    }
}
abstract class A{
    public int a = 10;
    public void hello() {
        System.out.println("hello!");
    }
    abstract public void move();
}

abstract class C extends A{
// 不用实现 move 方法

}
```
另外，由于抽象类必须要被继承，因此类的修饰中不能使用关键字 `final`，这是与继承矛盾的。
## 二、抽象类中的构造方法
抽象类中的构造方法一定是要有的，因为这是给子类实例化用的。一般默认为 `super()`，这点相当关键。

```java
public class Test01 {
    public static void main(String[] args) {
        B b = new B();
        System.out.println(b.a);
        b.hello();
        b.move();
    }
}
abstract class A{
	// 如果写了有参构造，那么再实例化子类变量时，就得用有参构造初始化，不然会报错.
    public A(){}
    public A(int a) {
        this.a = a;
    }
    public int a = 10;
    public void hello() {
        System.out.println("hello!");
    }
    abstract public void move();
}
class B extends A{
	// 默认构造方法为 super()
    @Override
    public void move() {
        System.out.println("Move!");
    }
}
```
## 三、接口
接口，英文称作`interface`，在软件工程中，接口泛指供别人调用的方法或者函数。从这里，我们可以体会到Java语言设计者的初衷，它是对行为的抽象，可以理解为插拔器件。
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/2fe0530c76674db49df1a4bf8a648f7c_1720253850025.png?token=ANB4BCPSTYGUWZIX4HKWSZTGRD66G)

接口与抽象类最大的区别在于，接口是完全抽象的，它里面只包含两个内容：常量和抽象方法，并且访问控制一定是 `public `的.

```java
public class JiekouTest01 {
    public static void main(String[] args) {
    }
}
interface A{
    public static final int a =  10;
    public abstract void move();
}
interface C{

}
class B implements A,C{
    @Override
    public void move() {
        System.out.println("move!");
    }
}
```
上面提到对于常量和抽象方法，它们的访问控制一定是 `public `的，因此对于常量，可以把`public static final`省略掉；对于方法，可以把`public abstract`省略掉。

也就是说，接口中的变量会被隐式地指定为`public static final`变量（并且只能是`public static final`变量，用`private`修饰会报编译错误），而方法会被隐式地指定为`public abstract`方法且只能是`public abstract`方法（用其他关键字，比如`private`、`protected`、`static`、 `final`等修饰会报编译错误。并且接口中所有的方法不能有具体的实现，接口中的方法必须都是抽象方法。从这里可以隐约看出接口和抽象类的区别，接口是一种极度抽象的类型，它比抽象类更加“抽象”，并且一般情况下不在接口中定义变量。
## 四、接口的继承和实现
接口作为一种抽象类，其实在实现的时候，就已经做到了继承。

```java
public class AbstractTest01 {
    public static void main(String[] args) {
        A a = new D();
        B b = new D();
        C c = new D();

        a.ma();
        b.mb();
        c.mc();
    }
}
interface A{
    void ma();

}
interface B{
    void mb();
}
interface C{
    void mc();

}
class D implements A,B,C{
    @Override
    public void ma() {
        System.out.println("ma!");
    }
    @Override
    public void mb() {
        System.out.println("mb!");
    }

    @Override
    public void mc() {
        System.out.println("mc!");
    }
}
```

可以看出，允许一个类遵循多个特定的接口。如果一个非抽象类遵循了某个接口，就必须实现该接口中的所有方法。但是对于遵循某个接口的抽象类，可以不实现该接口中的抽象方法。

接口与接口之间还可以强制转型，只要是接口，一定能转型成功，即在编译的时候不会报错，但是在运行的时候有可能会抛出异常。比如对于：

```java
	B b2 = new D();
    A a2 = (A)b2;
    a2.ma();
```
我们可以把B类型的对象`b2`转成`A`类型，由于 `D` 同时实现了 `A`，`B`，`C`接口，因此可以运行成功，但是如果`D` 类没有实现 `E`类型的接口，虽然可以编译不报错，但是运行就会抛异常。当然，也没人会这么写代码。

## 五、抽象类和接口的区别
### （一）语法层面上的区别
- 抽象类可以提供成员方法的实现细节，而接口中只能存在public abstract 方法；
- 抽象类中的成员变量可以是各种类型的，而接口中的成员变量只能是public static final类型的；
- 接口中不能含有静态代码块以及静态方法，而抽象类可以有静态代码块和静态方法；
- 一个类只能继承一个抽象类，而一个类却可以实现多个接口。

### （二）设计层面的区别
先写一个经典的例子：

```java
public class Test01 {
    public static void main(String[] args) {
        Birds bird = new Birds();
        Fish fish = new Fish();
        bird.color = "Blue";
        bird.sing();
        bird.fly();
        fish.fly();
    }
}
abstract class Birddd{
    public String color;
    abstract void sing();
}
interface flyable {
    void fly();
}
class Birds extends Birddd implements flyable{

    @Override
    public void fly() {
        System.out.println("Birds are flyable!");
    }
    @Override
    void sing() {
        System.out.println("Birds can sing!");
    }
}
class Fish implements flyable{
    @Override
    public void fly() {
        System.out.println("鱼插上翅膀也能飞!");
    }
}

```
在这个例子中，可以看出，**继承是一个 "是不是"的关系，而 接口实现则是 "有没有"的关系**。如果一个类继承了某个抽象类，则子类必定是抽象类的种类，而接口实现则是有没有、具备不具备的关系，比如鸟是否能飞（或者是否具备飞行这个特点），能飞行则可以实现这个接口，不能飞行就不实现这个接口。

*全文完，感谢你的阅读。*


