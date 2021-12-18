---
layout:     post
title:      FrameWork基础
subtitle:   反射 & 代理
date:       2021-12-18
author:     Ann
header-img: img/bg_eden.jpeg
catalog: true
tags:
    - framework
    - Android
    - Java
---

## 反射
> 反射包含以下技术
- 根据一个String得到一个类对象
- 根据一个类，获取类中字段、方法、属性
- 对泛型反射

### 获取类的方式
- 对象.getClass();
- Class.forName("对象全限定名");
- class属性  Class x = String.class;
- TYPE类型（装箱类） Class x = Boolean.TYPE;

### 反射方法
#### 获取类构造函数（Constructor）
- 获取类中所有构造函数 `Class.getDeclaredConstructors`
    ```java
    // 获取某个对象类实例
    Class temp = r.getClass();
    Constructor[] cons = temp.getDeclaredConstructors();
    for (Constructor c : cons) {
        // 获得修饰域
        int mod = c.getModifiers();
        // 获得构造方法参数集合
        Class[] parameterType = c.getParameterTypes();
    } 
    ```
- 通过反射执行类构造函数
    - 无参构造方法可通过Class.newInstance()/Constructor.newInstance()方法实现
    - 有参构造方法：
    ```java
    Class[] p3 = {int.class,String.class};
    Constructor ctor = r.getDeclaredConstructor(p3);
    Object obj = ctor.newInstance(1,"test");
    ```

#### 反射获取类变量 (Field)
- 获取类中的变量（Field对象）
    - Class.getDeclaredField("变量名称")
- 修改类中变量/静态变量
    ```java
    // 获取名为name的私有变量,temp为某个class对象
    Field nameField = temp.getDeclaredFiele("name");
    // 设置此方法可以被修改
    nameField.setAccessible(true);
    Object fieldObject = field.get(temp);
    nameField.set(fieldObject,"anan");

    // 获取名为name的静态变量,temp为某个class对象
    Field nameField = temp.getDeclaredFiele("name");
    // 设置此方法可以被修改
    nameField.setAccessible(true);
    Object fieldObject = field.get(null);
    nameField.set(fieldObject,"anan");

    ```

#### 反射获取类方法（Method)
- 获取类中方法/静态方法 （Method对象）
    - 有参方法：Class.getDeclaredMethod("方法名称",参数list);
    - 无参方法：Class.getDeclaredMethod("方法名称");
- 调用类中方法/静态方法
    ```java
    // 获取名为sum的有参方法,temp为某个class对象
    Class[] p2 = {int.class,int.class};
    Method sum = temp.getDeclaredMethod("sum",p2);
    // 设置此方法可以被修改
    sum.setAccessible(true);
    // 参数
    Object[] params = {1,2};
    sum.invoke(temp,params);

    // 获取名为get的无参静态方法,temp为某个class对象
    Method get = temp.getDeclaredMethod("get");
    // 设置此方法可以被修改
    get.setAccessible(true);
    // 静态方法不用指定具体对象
    get.invoke();
    ```

## 代理
> 代理模式是结构型设计模式，为其他对象提供一个代理以控制对某个对象的访问。代理类负责为委托类`预处理消息，过滤消息并转发消息`，以及进行`消息被委托类执行后的后续处理`。
![](https://img-blog.csdn.net/20140701192620959?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGVqaW5neXVhbjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
### 静态代理
    ```java
    public class UserManagerImplProxy implements UserManager {  
        // 目标对象  
        private UserManager userManager;  
        // 通过构造方法传入目标对象  
        public UserManagerImplProxy(UserManager userManager){  
            this.userManager=userManager;  
        }  
        @Override  
        public void addUser(String userId, String userName) {  
            try{  
                    //添加打印日志的功能  
                    //开始添加用户  
                    System.out.println("start-->addUser()");  
                    userManager.addUser(userId, userName);  
                    //添加用户成功  
                    System.out.println("success-->addUser()");  
                }catch(Exception e){  
                    //添加用户失败  
                    System.out.println("error-->addUser()");  
                }  
        }  
    
        @Override  
        public void delUser(String userId) {  
            userManager.delUser(userId);  
        }  
    
        @Override  
        public String findUser(String userId) {  
            userManager.findUser(userId);  
            return "张三";  
        }  
    
        @Override  
        public void modifyUser(String userId, String userName) {  
            userManager.modifyUser(userId,userName);  
        }  
    }  
    ```
### 动态代理
通过`Proxy.newProxyInstance()`方法完成动态反射！他可以套在任何一个接口类型的对象上，完成代理工作。
- newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler H) 
    - loader: 目标对象的类加载器
    - interfaces: 目标对象的接口类型
    - H: 通过此注入目标对象

    ```java
    //动态代理类只能代理接口（不支持抽象类），代理类都需要实现InvocationHandler类，实现invoke方法。该invoke方法就是调用被代理接口的所有方法时需要调用的，该invoke方法返回的值是被代理接口的一个实现类  
    public class LogHandler implements InvocationHandler {  
    
        // 目标对象  
        private Object targetObject;  
        //绑定关系，也就是关联到哪个接口（与具体的实现类绑定）的哪些方法将被调用时，执行invoke方法。              
        public Object newProxyInstance(Object targetObject){  
            this.targetObject=targetObject;  
            //该方法用于为指定类装载器、一组接口及调用处理器生成动态代理类实例    
            //第一个参数指定产生代理对象的类加载器，需要将其指定为和目标对象同一个类加载器  
            //第二个参数要实现和目标对象一样的接口，所以只需要拿到目标对象的实现接口  
            //第三个参数表明这些被拦截的方法在被拦截时需要执行哪个InvocationHandler的invoke方法  
            //根据传入的目标返回一个代理对象  
            return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(),  
                    targetObject.getClass().getInterfaces(),this);  
        }  
        @Override  
        //关联的这个实现类的方法被调用时将被执行  
        /*InvocationHandler接口的方法，proxy表示代理，method表示原对象被调用的方法，args表示方法的参数*/  
        public Object invoke(Object proxy, Method method, Object[] args)  
                throws Throwable {  
            System.out.println("start-->>");  
            for(int i=0;i<args.length;i++){  
                System.out.println(args[i]);  
            }  
            Object ret=null;  
            try{  
                /*原对象方法调用前处理日志信息*/  
                System.out.println("satrt-->>");  
                
                //调用目标方法  
                ret=method.invoke(targetObject, args);  
                /*原对象方法调用后处理日志信息*/  
                System.out.println("success-->>");  
            }catch(Exception e){  
                e.printStackTrace();  
                System.out.println("error-->>");  
                throw e;  
            }  
            return ret;  
        }  
    }  
    ```

### some thing want to say
> 关于代理模式，常与外观模式、适配器模式的概念混淆，其实他们侧重点是不一致的，在真正的代码设计中，往往是多种设计模式的组合，作为结构型的设计模式，以实际开发为例子难免会有混淆的情况，记录一下。[参考博客](https://blog.csdn.net/u010375663/article/details/35797163)
- 代理模式：作用于单个对象，主旨思想是给对象提供代理来控制方法。（保证原方法的稳定性，同时方便调用方做些定制化逻辑）。
- 适配器模式：作用于单个对象，主旨思想是适配桥接某个类方法为统一的接口模式。
- 外观模式：作用于多个对象，在独立类上抽取共有方法。以接口的设计方式，方便调用方通过统
一接口切换不同的系统。（SDK基础模块的思想）