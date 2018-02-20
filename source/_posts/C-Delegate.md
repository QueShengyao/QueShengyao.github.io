---
title: 'C# 委托'
date: 2018-02-20 11:44:09
tags:
---
# 委托是什么
在C语言中，有一个函数指针的概念：
`returnType (*func)(T1,  T2...)`

在这样一个语句中，我们定义了指向返回类型是returnType，参数为T1，T2...的函数的指针func，func实质上是一个指向函数对应内存地址的指针。
C#中的委托和函数指针概念很相近，它可以被理解为是.NET中类型安全的函数指针。委托允许我们在运行时指定调用的方法，但是该方法必须符合一定的规范（与所定义委托的返回类型和参数相同）。
实际上，.NET是在C语言函数指针的基础上为函数添加了一个容器（本文中的委托），只有符合这个容器尺寸（返回值，参数列表相同）的函数才能被装进这个容器。在Runtime，.NET会通过调用 容器代码来达成C语言中直接调用函数指针的目的。这也是委托为什么是类型安全的原因。
有趣的是，委托只检查传入函数的返回值和参数，并不关注传入函数是实例方法还是静态方法，它能接收这两种函数类型。
# 委托的使用
## 1.声明
在使用委托之前，我们首先需要声明它。
委托在定义时和关键字的使用方法类似，我们只需要在声明函数的语句前面加一个delegate，就把这个语句变成了委托声明语句，比如:
delegate double TwoLongsOp(long first, long second);

定义了一个输入两个long参数，返回一个double值的委托 TwoLongsOp。
## 2.使用
要想使用委托，我们还需要定义一个满足如上输入输出的函数，比如：
```
class MathOperations{
      double static TwoLongsAdd( long first, long second){
             return first + second;
        }
}
```
这里定义一个静态方法，然后，定义一个委托实例：
`TwoLongsOp  addOp = MathOperations.TwoLongsAdd; `

这样就完成了一个委托的初始化，addOp现在是一个引用MathOperations类中静态方法TwoLongsAdd，我们可以在代码中使用这个委托调用TwoLongsAdd方法了：
```
static invokeDelegate(TwoLongsOp op, double first, double second){
       Console.WriteLine($"LoingOperation Result is : {0}", op(first, second));
}
invokeDelegate(addOp, 1.0, 2.0);
```
以上代码的运行结果是“LoingOperation Result is : 3.0 ”
## 3.Action<T> 和 Func <T>
这是C#的语法糖，它们的作用在于更简洁地声明和使用委托，节约代码空间和程序员时间。使用它们，我们就可以省去之前的声明和赋值步骤了：
```
static invokeDelegate(Func<double, double,. double> op , double first, doube second){
    Console.WriteLine($"LoingOperation Result is : {0}", op(first, second));
}
invokeDelegate(MathOperat ions.TwoLongsAdd, 1.0, 2.0);
```
这段代码和以上的三段加在一起实现的功能是等价的，这让委托的语法变得更简洁了。
## 4.多播委托
和函数指针显著不同的一点是，.NET允许我们包含多个函数调用，只需要调用一个委托，就可以一次执行这个委托中包含的所有方法，我们称这个特性为多播委托。
需要注意的是，多播委托引用的函数的返回类型必须是void，否则，我们只能得到最后一个函数调用结果。具体的例子可以参考C#高级编程或者是msdn文档。
# 委托的实现
事实上，当我们声明一个委托时，实际上定义了一个新类，一个派生于System.MulticastDelegate 的类，而System.MulticastDelegate 又派生于System.Delegate。这两个类很有意思，.NET能够创建派生于它们的类，程序员不能手动创建它们。如果你尝试手动继承一个Delegate或者MulticastDelegate类，会看到这样的错误信息：
`Error CS0644:'XXXXClass' cannot derive from special class 'MulticastDelegate' `

Delegate 和 MulticastDelegate被.NET定义为特殊类，我们无法创建一个派生自特殊类的自定义类。
观察C#的源码（[http://source.roslyn.codeplex.com/#Microsoft.CodeAnalysis/SpecialType.cs,5b11a29d644330dc](http://source.roslyn.codeplex.com/#Microsoft.CodeAnalysis/SpecialType.cs,5b11a29d644330dchttp://source.roslyn.codeplex.com/#Microsoft.CodeAnalysis/SpecialType.cs,5b11a29d644330dc)），我们可以发现，微软用一个enum类型SpecialType直接hardcode了所有的特殊类，实现方式相当巨硬。。。