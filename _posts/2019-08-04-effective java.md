---
layout:     post
title:      《effective java》学习笔记
subtitle:   创建和销毁对象
date:       2019-08-04
author:     Ann
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 学习笔记
    - java
---

### 第一条 用静态工厂方法代替构造器
- 什么是静态工厂方法？
譬如
```java 
public static Integer valueOf(int i) {
    assert IntegerCache.high >= 127;
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
Integer i = Integer.valueOf(123);
```

这种形式的，不通过 new，而是用一个静态方法来对外提供自身实例的方法，即为我们所说的静态工厂方法(Static factory method)。

> new做了什么？  
当我们使用 new 来构造一个新的类实例时，其实是告诉了 JVM 我需要一个新的实例。JVM 就会自动在内存中开辟一片空间，然后调用构造函数来初始化成员变量，最终把引用返回给调用方。

- 静态工厂方法相比较于普通构造器的优势
    - 静态工厂可以给实例化的对象起名字，更方便解释其意义
    - 每次调用时不用创建一个新对象（如valueOf）
    - 可以返回原返回类型的任何子类型对象

- 静态工厂方法缺点
    - 类若不含有public或protected构造器，无法被子类化
        - 强制不使用继承，只能用组合

### 第二条 遇到多个构造器参数时考虑用Builder

静态工厂和构造器共同的局限性——<strong>无法很好的扩展到大量的可选参数</strong>

- JavaBean
    - 调用一个无参构造函数来创建对象，然后通过setter方法来设置必要参数
    - <font color="red">缺点：</font>构造过程被分割到好多个set中容易造成线程不安全，导致对象处于不一致的状态

- Builder
    - 不直接生成想要的对象，通过调用构造器（或静态工厂），得到builder对象，客户端在builder对象上调用类似setter方法，来设置每个相关的可选参数，最后，再调用build方法来生成不可变对象

例如：
构造类：
```java
public class Person {
    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", address='" + address + '\'' +
                ", firstName='" + firstName + '\'' +
                ", lastName='" + lastName + '\'' +
                '}';
    }

    private String name;
    private String address;
    private String firstName;
    private String lastName;

    private Person(String name, String address, String firstName, String lastName) {
        this.address = address;
        this.name = name;
        this.lastName = lastName;
        this.firstName = firstName;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {

        private String name;
        private String address;
        private String firstName;
        private String lastName;

        public Builder name(final String name) {
            this.name = name;
            return this;
        }

        public Builder address(final String address) {
            this.address = address;
            return this;
        }

        public Builder lastName(final String lastName) {
            this.lastName = lastName;
            return this;
        }

        public Builder firstName(final String firstName) {
            this.firstName = firstName;
            return this;
        }

        public Person build() {
            return new Person(name, address, firstName, lastName);
        }
    }
}
```
客户端调用：
```java
        Person p = new Person.Builder().address("上海").firstName("三").lastName("张").name("欧阳").build();
        System.out.println(p);
        Person person = Person.builder().address("上海").firstName("三").lastName("张").name("欧阳").build();
        System.out.println(person);
```


<strong><font color="red">Lombok</font></strong>----传统的builder定义方法有些复杂，Lombok是一个可以让Java代码变的更加简洁、让你的开发更加高效的利器。使用了Lombok之后，我们不需要写Getter&Setter、ToString等方法，这些都可以通过注解来代替，在编译期间，Lombok会帮助你生成相应的字节码。所以也不用担心性能损失。

Lombok也支持了Builder模式，你可以用几个注解来代替以上冗余的代码。

```java
@Builder
public class Order {
    private String code;
    @Singular
    private List<String> offers;
    @Singular
    private Map<String, Object> features;
}
```

客户端调用

```java
Order order = Order.builder().code("1234")
   .offer("满100减5")
   .feature("category", "category")
   .build();
```

- 总结
    - 创建对象，必须先创造构建器，造成一定的性能上的开销
    - 如果类的构造器或静态工厂中具有多个参数，可以考虑用Builder
    - 当无法预知该类的参数会不会增加的时候，最好用Builder模式

### 第三条 用私有构造器或者枚举类型来强化singleton属性
>Singleton(单例）指仅仅被实例化一次的类.
我们需要定义一个private的构造器，但要注意一点，即使我们定义了私有的构造器，但是客户端还是可以借助AccessibleObject.setAccessible方法，通过反射来调用私有的构造器，因此，我们需要修改构造器来抵御这种工具，下面代码很好阐述了这个。

```java
public class Singleton {
        private static final Singleton INSTANCE = new Singleton();

        private Singleton() {
            if (INSTANCE != null) {
                throw new UnsupportedOperationException("Instance already exist");
            }
        }

        public static Singleton getInstance() {
            return INSTANCE;
        }
}
```

然而这样做就可以绝对防止出现多个实例了么？其实现在还有一种情况下会出现多个实例，那就是在你序列化这个对象之后，在进行反序列化，这个时候，你将再次得到一个新的对象，让我们看下例子，首先实现序列化接口

```java
public class SerializableTest {

      //序列化
      private static  void serializable(Singleton singleton, String filename) throws IOException {
          FileOutputStream fos = new FileOutputStream(filename);
          ObjectOutputStream oos = new ObjectOutputStream(fos);
          oos.writeObject(singleton);
          oos.flush();
      }
      
      //反序列化
      @SuppressWarnings("unchecked")
      private static <T> T deserializable(String filename) throws IOException,
                                              ClassNotFoundException {
          FileInputStream fis = new FileInputStream(filename);
          ObjectInputStream ois = new ObjectInputStream(fis);
          return (T) ois.readObject();
      }
      
      public static void main(String[] args) throws IOException, ClassNotFoundException {
        //序列化到文件test.txt中
        serializable(Singleton.getInstance(),"F://test.txt");
        //反序列化
        Singleton singleton = deserializable("F://test.txt");
        
        //比较反序列化后得到的singleton和Singleton.getInstance的地址是否一样。
        System.out.println(singleton);
        System.out.println(Singleton.getInstance());
    }
}
```
运行测试结果显示反序列化后得到的singleton和Singleton.getInstance的地址是不一样的，即创建了两边单例。

那么如何解决序列化问题呢，其实很简单，只要我们在Singleton中加入下面方法即可，为什么只需要加入<strong>readResolve</strong>就好了呢，因为任何一个readObject方法，不管显示还是默认的，它都会返回一个新建的实例，这就是为什么上面两个实例的地址是不一样的原因了，而加入了readResolve之后，那么在反序列化之后，新建对象上的readResolve方法会被调用，然后该方法返回的对象引用将取代新建的对象，指向新建对象的引用就不会保留下来，立即成为垃圾回收的对象了。

```java
 private Object readResolve() {
        return INSTANCE;
 }
```

<font color='pink'>利用枚举来强化Singleton，不会出现上面这些情况，Java1.5后，利用<strong><font color='red'>单元素</font></strong>的枚举来实现单例（Singleton），绝对防止多次实例化，实现代码如下：</font>

```java
public enum Elvis{
    INSTANCE;
    private String[] favoriteSongs = {"Hound Dog","Heartbreak Hotl"};
    public void printFavorites(){
         System.out.println(Arrays.toString(favoriteSongs ));
    }      
}
```
调用时只需要Elvis.INSTANCE.printFavorites();即可。


- 总结
    - 利用单元素的枚举类型已经成为实现单例的最佳方法

- Todo
    - 反射
    - 序列化与反序列化
    - 单例的几种实现形式
    - 什么情况下会用到单例？

### 第四条 通过私有构造器强化不可实例化的能力

- java中让类不可实例化主要通过三个方面
    - 抽象类
        - 虽然抽象类不可实例化，但可以被子类化，且子类可以实例化
    - 内部类
        - 内部类的实例化需要借助外部类
    - <strong>将构造函数的权限设为private</strong>
        - 构造函数私有，便不能被实例化，这种情况常见于官方提供的类中，例如Math类和System类。

- 为啥工具类要不可实例化？
    - 工具类是为了提供一些通用类的某一非业务领域内的公共方法，不需要配套的成员变量，仅仅是作为工具方法被使用。（无需实例化）
    - 同时，即使类里没有成员变量，实例化也是要分配内存的，那会消耗资源。尤其是在高并发的场景中，如果工具类需要实例化，且不是单例，那会消耗许多资源，导致gc频繁调用，影响程序性能。 （若实例化会增加系统负担）
    - [工具类用单例模式还是静态方法好？](https://my.oschina.net/u/2930289/blog/1923536)

### 第五条 避免创建不必要的对象

- 第一条建议中，用静态工厂方法代替构造器可以避免重复创建不必要的对象。
- 通过延迟初始化，可以去除一些不必要的初始化工作，但会使方法的实现更加复杂。<font color="red">可以，但没必要</font>
- <font color="red">建议：</font> 要优先使用基本类型而非装箱类型，自动装箱会创建若干新对象导致性能下降。
- <font color="red">合理运用对象池</font> 
    - 对象池维护困难，通过创建附加对象可以提升程序的清晰性，简洁性和功能性是一件好事
    - 对于数据库，构建数据库连接池是很有必要的，因为数据库连接的代价很高。