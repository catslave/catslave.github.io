---
layout: post
title: Design Pattern
category: Design Pattern
description: 常见设计模式
---

## 创建型模式

### 单例模式

单例模式保证系统实例的唯一性。首先需要将构造方法私有化，避免外部程序创建该类实例。然后提供一个全局静态方法获取该类实例，用户也只能通过该方法获取实例。
> 1. 构造函数私有化
> 2. 提供一个方法获取实例（因为外部不能创建该类对象，所以提供的方法只能是静态方法，以及提供的实例变量也是静态变量）

单例模式常见实现：
1. 懒汉模式（要用的时候才创建实例）
2. 饿汉模式（类加载的时候就创建实例）
3. 静态内部类（类加载的时候就创建实例）
4. 双重锁校验

#### 懒汉模式


    public class LazySingleton {

        // 这里必须是私有静态变量
        private static LazySingleton instance;

        // 1. 构造函数私有化
        private LazySingleton() {}

        // 2. 提供一个方法获取实例
        public static synchronized LazySingleton getInstance() {
            if(instance == null) {
                instance = new LazySingleton();
            }
            return instance;
        }
    }

#### 饿汉模式

    public class HungrySingleton {

        // 类加载时候就创建实例
        private static HungrySingleton instance = new HungrySingleton();

        private HungrySingleton() {

        }

        public static HungrySingleton getInstance() {
            return instance;
        }
    }

##### 静态内部类

    public class Singleton {

        static class SingletonHolder() {
            // 静态内部类来创建外部类实例，在类加载过程的初始化阶段，就会创建实例
            private static final Singleton instance = new Singleton();
        }

        private Singleton() {}

        public static final Singleton getInstance() {
            return SingletonHodler.instance;
        }
    }

##### 双重锁校验

    public class LazySingleton {

        // 这里必须是私有静态变量
        private volatile static LazySingleton instance;

        // 1. 构造函数私有化
        private LazySingleton() {}

        // 2. 提供一个方法获取实例
        public static LazySingleton getInstance() {
            if(instance == null) {
                synchronized(LazySingleton.class) {
                    if(instance == null) {
                        instance = new LazySingleton();
                    }
                }
            }
            return instance;
        }
    }

##### 反射问题

Class对象的Constructor.newInstance()方法可以调用私有的构造函数创建对象，即可以通过反射获取对象实例。
> 反射可以获取类中的方法、构造函数，并修改它们的访问权限。

这里以HungrySingleton举例（LazySingleton一样）

    public class TestSingleton {


        public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {

            HungrySingleton instance = HungrySingleton.getInstance();

            System.out.println(instance);

            Constructor<HungrySingleton> constructor = HungrySingleton.class.getDeclaredConstructor();

            // 这里通过反射又实例化出了一个新的对象
            constructor.setAccessible(true);
            HungrySingleton hungrySingleton = constructor.newInstance();

            System.out.println(hungrySingleton);
        }
    }


可以给构造函数做条件判断，判断静态变量是否已初始化，如果初始化过就直接抛异常。但是这种方式也会有问题，如果先通过反射创建对象，那对于懒汉模式的静态变量还是null，后面再通过getInstance方法又会创建一个新的实例处理。

##### 枚举实现单例

### 原型模式

调用实例的clone方法来创建对象。

Java中的复制分为浅复制和深复制：
- 浅复制：不复制对象的引用类型，基本数据类型重新创建并赋值，引用类型直接指向原有对象的引用。
- 深复制：基本类型和引用类型都重新创建和复制。通过重新引用类型的clone方法，实现深拷贝。

### 代理模式

通过代理对象实现对原对象方法的访问。
> - 原本方式：直接实例化实现了目标接口的目标类对象
> - 代理方法：实例化实现了目标接口的代理类对象，由代理对象内部创建并持有目标类对象，并代理目标对象的方法调用。

1. 代理和对象都实现目标接口；
2. 代理并持有对象，在方法内部调用对象。

### 外观模式

外观模式也叫门面模式，Facade类对目标类的细节进行屏蔽，对外只提供必需的接口和数据。

Tomcat与Servlet的交互利用了门面模式。

## 结构型模式

### 桥接模式


## 行为型模式

### 策略模式

为同一个行为定义不同策略，并为每个策略都实现了不同的方法，系统根据不同策略自动切换行为。

策略模式的实现是在接口中定义不同的策略，由实现类完成策略的具体实现，并为用户提供一个策略状态存储**上下文（Context）**中完成策略的存储和状态的改变。
> 1. 定义策略接口
> 2. 实现多个策略接口实现类
> 3. 定义策略上下文，持有所有策略实现类通过用户传入策略ID获取对应策略实现（选择策略逻辑由上下文实现），或者用户直接传入策略实现类（选择策略逻辑由用户实现）

### 模板模式

父类实现公共部分算法，子类实现可变部分的算法。父类一般定义为抽象类，实现基础代码，提供抽象方法，将具体实现延迟到子类中。
> 1. 定义抽象类：实现基础代码（内部调用抽象模板方法），提供抽象模板方法
> 2. 子类继承抽象类：实现抽象模板方法，该方法由父类调用