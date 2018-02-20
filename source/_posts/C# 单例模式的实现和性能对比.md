---
title: 'C# 单例模式的实现和性能对比'
date: 2018-02-20 11:45:07
tags:
---
## 简介
单例指的是**只能存在一个实例的类**（在C#中，更准确的说法是在每个AppDomain之中只能存在一个实例的类，它是软件工程中使用最多的几种模式之一。在第一个使用者创建了这个类的实例之后，其后需要使用这个类的就只能使用之前创建的实例，无法再创建一个新的实例。通常情况下，单例会在第一次被使用时创建。本文会对C#中几种单例的实现方式进行介绍，并分析它们之间的线程安全性和性能差异。

单例的实现方式有很多种，但从最简单的实现（非延迟加载，非线程安全，效率低下），到可延迟加载，线程安全，且高效的实现，它们都有一些基本的共同点：

    . 单例类都只有一个private的无参构造函数
    . 类声明为sealed（不是必须的）
    . 类中有一个静态变量保存着所创建的实例的引用
    . 单例类会提供一个静态方法或属性来返回创建的实例的引用（eg.GetInstance）

## 几种实现
### 一. 非线程安全
```csharp
//Bad code! Do not use!
public sealed class Singleton
{
    private static Singleton instance = null;

    private Singleton()
    {

    }

    public static Singleton instance
    {
        get
        {
            if (instance == null)
            {
                instance = new Singleton();
            }
            return instance;
        }
    }
}
```
这种方法不是线程安全的，会存在两个线程同时执行`if (instance == null)`并且创建两个不同的instance，后创建的会替换掉新创建的，导致之前拿到的reference为空。

### 二. 简单的线程安全实现
```csharp
public sealed class Singleton
{
    private static Singleton instance = null;
    private static readonly object padlock = new object();

    Singleton()
    {
    }

    public static Singleton Instance
    {
        get
        {
            lock (padlock)
            {
                if (instance == null)
                {
                    instance = new Singleton();
                }
                return instance;
            }
        }
    }
}
```
相比较于实现一，这个版本加上了一个对instance的锁，在调用instance之前要先对padlock上锁，这样就避免了实现一中的线程冲突，该实现自始至终只会创建一个instance了。但是，由于每次调用Instance都会使用到锁，而调用锁的开销较大，这个实现会有一定的性能损失。

注意这里我们使用的是新建一个private的object实例padlock来实现锁操作，而不是直接对Singleton进行上锁。直接对类型上锁会出现潜在的风险，因为这个类型是public的，所以理论上它会在任何code里调用，直接对它上锁会导致性能问题，甚至会出现死锁情况。

**Note**: C#中，同一个线程是可以对一个object进行多次上锁的，但是不同线程之间如果同时上锁，就可能会出现线程等待，或者严重的会出现死锁情况。因此，我们在使用lock时，尽量选择类中的私有变量上锁，这样可以避免上述情况发生。

### 三. 双重验证的线程安全实现
```csharp
public sealed calss Singleton
{
    private static Singleton instance = null;
    private static readonly object padlock = new object();

    Singleton()
    {
    }

    public static Singleton Instance
    {
        get
        {
            if (instance == null)
            {
                lock (padlock)
                {
                    if (instance == null)
                    {
                        instance = new Singleton();
                    }
                }
            }
            return instance;
        }
    } 
}
```
在保证线程安全的同时，这个实现还避免了每次调用Instance都进行lock操作，这会节约一定的时间。

但是，这种实现也有它的缺点：

    1. 无法在Java中工作。（具体原因可以见原文，这边没怎么理解）
    2. 程序员在自己实现时很容易出错。如果对这个模式的代码进行自己的修改，要倍加小心，因为double check的逻辑较为复杂，很容易出现思考不周而出错的情况。

### 四. 不用锁的线程安全实现
```csharp
public sealed class Singleton
{
    //在Singleton第一次被调用时会执行instance的初始化
    private static readonly Singleton instance = new Singleton();

    //Explicit static consturctor to tell C# compiler 
    //not to mark type as beforefieldinit
    static Singleton()
    {
    }

    private Singleton()
    {
    }

    public static Singleton Instance
    {
        get
        {
            return instance;
        }
    }
}
```
这个实现很简单，并没有用到锁，但是它仍然是线程安全的。这里使用了一个static，readonly的Singleton实例，它会在Singleton第一次被调用的时候新建一个instance，这里新建时候的线程安全保障是由.NET直接控制的，我们可以认为它是一个原子操作，并且在一个AppDomaing中它只会被创建一次。

这种实现也有一些缺点:

    1. instance被创建的时机不明，任何对Singleton的调用都会提前创建instance
    2. static构造函数的循环调用。如有A，B两个类，A的静态构造函数中调用了B，而B的静态构造函数中又调用了A，这两个就会形成一个循环调用，严重的会导致程序崩溃。
    3. 我们需要手动添加Singleton的静态构造函数来确保Singleton类型不会被自动加上beforefieldinit这个Attribute，以此来确保instance会在第一次调用Singleton时才被创建。
    4. readonly的属性无法在运行时改变，如果我们需要在程序运行时dispose这个instance再重新创建一个新的instance，这种实现方法就无法满足。

### 五. 完全延迟加载实现（fully lazy instantiation）
```csharp
public sealed class Singleton
{
    private Singleton()
    {
    }

    public static Singleton Instance 
    {
        get
        {
            return Nested.instance;
        }
    }

    private class Nested
    {
        // Explicit static constructor to tell C# compiler
        // not to mark type as beforefieldinit
        static Nested()
        {
        }

        internal static readonly Singleton instance = new Singleton();
    }
}
```
实现五是实现四的包装。它确保了instance只会在Instance的get方法里面调用，且只会在第一次调用前初始化。它是实现四的确保延迟加载的版本。

### 六 使用.NET4的Lazy<T>类型
```csharp
public sealed class Singleton
{
    private static readonly Lazy<Singleton> lazy = new Lazy<Singleton>(() => new Singleton());

    public static Singleton Instance 
    {
        get 
        {
            return lazy.Value;
        }
    }

    private Singleton()
    {
    }
}
```
.NET4或以上的版本支持Lazy<T>来实现延迟加载，它用最简洁的代码保证了单例的线程安全和延迟加载特性。

## 性能差异
之前的实现中，我们都在强调代码的线程安全性和延迟加载。然而在实际使用中，如果你的单例类的初始化不是一个很耗时的操作或者初始化顺序不会导致bug，延迟初始化是一个可有可无的特性，因为初始化所占用的时间是可以忽略不计的。

在实际使用场景中，如果你的单例实例会被频繁得调用（如在一个循环中），那么为了保证线程安全而带来的性能消耗是更值得关注的地方。

为了比较这几种实现的性能，我做了一个小测试，循环拿这些实现中的单例9亿次，每次调用instance的方法执行一个count++操作，每隔一百万输出一次，运行环境是MBP上的Visual Studio for Mac。结果如下：

||线程安全性|延迟加载|测试运行时间(ms)|
|--------------|-------------|-----|---|
| 实现一 | 否 | 是 | 15532 |
| 实现二 | 是 | 是 | 45803 |
| 实现三 | 是 | 是 | 15953 |
| 实现四 | 是 | 不完全 | 14572 |
| 实现五 | 是 | 是 | 14295 |
| 实现六 | 是 | 是 | 22875 |

测试方法并不严谨，但是仍然可以看出，方法二由于每次都需要调用lock，是最耗时的，几乎是其他几个的三倍。排第二的则是使用.NET Lazy类型的实现，比其他多了二分之一左右。其余的四个，则没有明显区别。

## 总结
总体来说，上面说的多种单例实现方式在现今的计算机性能下差距都不大，除非你需要特别大并发量的调用instance，才会需要去考虑锁的性能问题。

对于一般的开发者来说，使用方法二或者方法六来实现单例已经是足够好的了，方法四和五则需要对C#运行流程有一个较好的认识，并且实现时需要掌握一定技巧，并且他们节省的时间仍然是有限的。

## 引用
本文大部分是翻译自[Implementing the Singleton Pattern in C#](http://csharpindepth.com/Articles/General/Singleton.aspx)，加上了一部分自己的理解。这是我搜索[static readonly field initializer vs static constructor initialization](https://stackoverflow.com/questions/2761393/static-readonly-field-initializer-vs-static-constructor-initialization)时看到的，在这里对两位作者表示感谢。
