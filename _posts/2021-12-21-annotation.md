---
layout:     post
title:      注解
subtitle:   
date:       2021-12-21
author:     Ann
header-img: img/bg_home.jpeg
catalog: true
tags:
    - Java 
---
> 看到很多代码里都用了注解，尤其是jectpack的一些新组件，都用了注解的方式，一直没闹明白注解如何在代码中作用的，学习记录一下。

![注解](http://openfile.shixinke.com/images/posts/2019/01/b601c6bdb38c14482515efd931767fd0.png)

## 注解概念&作用
- 什么是注解

源代码中元数据的一种标记，注解本质上是一个继承自Annotation的类(一般通过反射的方式实现具体的功能)

- 注解的作用
    - 生成文档，根据文档注解，可以生成java文档
    - 追踪代码依赖性，实现替代配置文件功能(最主要的功能)
    - 在编译时进行格式检查，告知编译器哪些代码需要检查

## 常见注解
### jdk注解
> jdk自带的注解

- @Override：告诉编译器我重写了接口方法
- @Deprecated：告诉编译器这个方法过时了，不建议使用，Ide会在方法上划横线
- @SuppressWarnings("deprecation"):关闭方法中出现的警告

### 元注解
> 用来描述注解的注解

#### @Target
说明了Annotation被修饰的范围，可被用于 packages、types（类、接口、枚举、Annotation类型）、类型成员（方法、构造方法、成员变量、枚举值）、方法参数和本地变量（如循环变量、catch参数）。在Annotation类型的声明中使用了target可更加明晰其修饰的目标

> 例：  
@Target(ElementType.TYPE)  
1.ElementType.CONSTRUCTOR:用于描述构造器  
2.ElementType.FIELD:用于描述域（类的成员变量）  
3.ElementType.LOCAL_VARIABLE:用于描述局部变量（方法内部变量）  
4.ElementType.METHOD:用于描述方法  
5.ElementType.PACKAGE:用于描述包  
6.ElementType.PARAMETER:用于描述参数  
7.ElementType.TYPE:用于描述类、接口(包括注解类型) 或enum声明  

#### @Retention
定义了该Annotation被保留的时间长短，有些只在源码中保留，有时需要编译成的class中保留，有些需要程序运行时候保留。即描述注解的生命周期

>例：  
@Retention(RetentionPolicy.RUNTIME)
1.RetentionPoicy.SOURCE:在源文件中有效（即源文件保留）  
2.RetentionPoicy.CLASS:在class文件中有效（即class保留）  
3.RetentionPoicy.RUNTIME:在运行时有效（即运行时保留）  


#### @Documented
它是一个标记注解，即没有成员的注解，用于描述其它类型的annotation应该被作为被标注的程序成员的公共API，因此可以被例如javadoc此类的工具文档化

#### @Inherited
它也是一个标记注解，它的作用是，被它标注的类型是可被继承的，比如一个class被@Inherited标记，那么一个子类继承该class后，则这个annotation将被用于该class的子类。

注意：一个类型被@Inherited修饰后，类并不从它所实现的接口继承annotation，方法并不从它所重载的方法继承annotation。

## 自定义注解
### 自定义注解
> public @interface 注解名 {定义体}

举个栗子：
```java
package main.java.com.shixinke.java.demo.annotation;
import java.lang.annotation.*;
@Target({ElementType.ANNOTATION_TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Check {
    boolean value() default false;
}
```
- 注意
    - 注解只有`public` 和 `default`两种访问修饰符
    - 参数成员只能用基本类型byte,short,char,int,long,float,double,boolean八种基本数据类型和 String,Enum,Class,annotations等数据类型，以及这一些类型的数组。
    - 如果只有一个参数成员,最好把参数名称设为"value",后加小括号。
    - 默认值用`default`关键字修饰

### 标记注解
在需要使用注解的位置上打入注解

```java
package main.java.com.shixinke.java.demo.annotation;
@Check(true)
public class User {
}
```

### 使用注解
注解本质只是一个注释，如果没有实现没有什么特殊含义，@Retention中，只有定义为RUNTIME的注解会被保留到运行状态，运行过程中，我们通过`反射`获取注解，并执行相关操作。

SOURCE状态的注解往往用于代码coding时期的格式检查，如@NonNull，自定义枚举注解等。

CLASS状态的注解貌似和SOURCE类似，打进class类中方便一些JNI的函数使用（我猜的 需要再查下资料

注解解析器来解析注解并实现功能

```java
package main.java.com.shixinke.java.demo.annotation;
import java.lang.annotation.Annotation;
public class CheckParser {
    public void parse(Class cls) {
        /**
         * 获取Check的注解对象
         */
        Annotation annotation = cls.getAnnotation(Check.class);
        Check check = (Check) annotation;
        /**
         * 检查是否有Check注解，并且注解的value属性为true
         */
        if (check != null && check.value()) {
            System.out.println(cls.getName()+"在检查中........");
        }
    }
}
```

调用注解解析器

``` java
package main.java.com.shixinke.java.demo.annotation;
public class AnnotationDemo {
    public static void main(String[] args) {
        User user = new User();
        Person person = new Person();
        CheckParser parser = new CheckParser();
        parser.parse(user.getClass());   //main.java.com.shixinke.java.demo.annotation.User在检查中........
        parser.parse(person.getClass());
    }
}
```

### 自定义注解demo
🚗🚗[lifecycle源码解析](https://blog.csdn.net/c10WTiybQ1Ye3/article/details/107852553?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-1.no_search_link&spm=1001.2101.3001.4242.2)