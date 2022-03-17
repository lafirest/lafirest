+++
title = "通过C#理解Monad"
date = 2021-07-12T02:40:30+08:00
tags = ["Monad"]
draft = false
+++

## 关于什么是 Monad {#关于什么是-monad}

讲解 Monad 是什么的文章很多，但是基本上都是以范畴论的概念来展开说明的,这导致了理解成本特别高 <br/>
从一个小坑掉入了一个大坑 <br/>
这里不谈理论，将通过下面的 C#(伪)代码来说明什么是 Monad，所以其实并不严谨 <br/>

```csharp
public delegate Gold F(int x);
public class Gold
{
    private int gold;

    public Gold(int x) => gold = x;

    public static Gold operator + (F f) => f(gold);

    public static Gold Double(int x) => new Gold(x * 2);
    public static Gold Zero(int x) => new Gold(0);
}
```


## 计算表达式 {#计算表达式}

微软在设计 F#时，为了避免使用 Monad 这个单词，发明了「计算表达式」这个概念，这个概念其实很好的反映了 Monad 的本质 <br/>
先来看个小学级别的计算式子: &alpha; = 1 + 2 + 3 + 4 + 5 <br/>
然后我们把里面的数字全部换成 Gold 类型，得到: &beta; = Gold + Gold + Gold + Gold + Gold <br/>
但是上面我们并没有实现 Gold + Gold, 而是实现了 Gold + F, 不过 Gold + F -&gt; Gold, 所以我们可以将 &beta; 里面的 Gold 替换为 Gold + F, <br/>
从而得到: &gamma; = (((Gold + F) + F) + F) + F <br/>


### 对 &gamma; 的扩展 {#对-and-gamma-的扩展}

1.  首先我们能看到，在上面的代码中，F可以是「Double」也可以是「Zero」,也就是这个表达式并不关心执行的具体流程，只要该流程满足 F 的声明即可 <br/>
2.  其次我们将表达式中的 Gold 换成 Diamond、HP、MP，并不影响 &gamma; 这个表达式的运算，也就是说这个表达式并不关心作用对象的类型 <br/>
3.  最后，我们可以发现 &gamma; 这个表达式并不关心 Gold 里面是 int gold 还是 long gold,或者 string gold, 也就是说这个表达式并不关心对象的内部环境 <br/>

总结: <br/>

-   我们将 1 中的执行流程用 F 表示 <br/>
-   2 中提到的对象用 M 表示 <br/>
-   3 中提到的对象内部环境用 a 表示 <br/>

那么得到新的表达式: &delta; = ((((M a) + F) + F) + F) <br/>
其中: F = a -&gt; M a <br/>


## 回到 Monad {#回到-monad}

上面的公式 &delta; 其实和 Haskell 中的 Monad 很类似了，Monad 可以看作是对公式 &delta; 这一类运算的抽象 <br/>
从 cshap 的角度理解, Monad 可以看作是一个接口或者抽象类, (伪)代码如下: <br/>

```csharp
public interface Monad<T<U>>
{
    Monad<T<U>> Return(U a);
    static Monad<T<U>> Bind (Monad<T<U>> m, Func<U, Monad<T<U>>> f)
}

```

其中: <br/>

-   U 是被包裹的类,比如 Gold 里面的 gold 的类型 int <br/>
-   其次 T 是外层的包裹类，比如 Gold <br/>
-   然后「Return」用于将一个被包裹的类提升为一个 Monad 的类 <br/>
-   然后 「Bind」则是 &delta; 中的 「+」一个二元运算 <br/>

这些基本上也是 Haskell 中实现一个 Monad 需要完成的"接口"实现(当然上面的代码在 C#里面是没法运行的,C#只能有限的模拟 Haskell 中的 moand) <br/>
也就是说只要有类和该类的行为，满足或者说实现这个接口，这个类就可以被看作是一个 Monad <br/>
所以，抛开理论上的定义，对 Monad 的使用可以看作是通过二元运算，串联起来的一连串的运算，而不同的二元运算可以实现不同的逻辑，从而达到对运行流程的控制、对副作用的管理等功能 <br/>
