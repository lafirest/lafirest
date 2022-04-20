+++
title = "Programming Language Roam: Van Laarhoven lens"
date = 2022-04-20T23:13:00+08:00
tags = ["fsharp"]
draft = false
+++

lens 是一种函数式引用，可以实现对数据的任意部分进行访问和修改，宽松点说，lens 可以看作是一种函数式「指针」。
现在最常用的 lens 实现是由 [Twan van Laarhoven](https://twanvl.nl/) 发明的 [CPS based functional references](https://www.twanvl.nl/blog/haskell/cps-functional-references)，原理虽然很简单，
但却相当惊艳。

这篇算是我自己的复习吧，将需要的概念都简单温习下。


## getter and setter {#getter-and-setter}

  任何一个可以读写的数据都至少有两个操作:读和写，对数据的读和写进行封装，以函数的方式对外提供的操作一般称作「getter」和「setter」。
之所以需要对读/写进行封装，是因为很多场景下，数据的写入往往伴随着一定的副作用，而对读进行封装，则可以实现延迟计算、单例模型
等功能。大部分现代编程语言，比如 C#，都在语法层面提供了简洁的 getter/setter 支持


## immutable object 和 函数式语言 {#immutable-object-和-函数式语言}

  在部分编程语言中存在不可变对象，比如在区分「值类型」和「引用类型」的编程语言中，值类型一般就是不可变的。对不可变对象
进行更新操作，一般都是先复制出一个新的副本，然后将更新的值作用在这个副本之上，从而得到更新后的对象。而在大多数函数式语言中，
任何对象几乎都是不可变的。

  不可变特性导致了一个很难受的问题：对复杂数据，尤其是层级很深的数据进行更新将是灾难性的，每次更新实际上都需要从当前层级
一直向上更新到根层级，这导致了更新的代码十分冗长。

所以有没有一种简单的方式，可以像 Cee 的指针，或者 C++ 的引用那样，对任意层级的任意数据进行访问和更新？

基于这种需求，lens 技术诞生了


## 最原始而朴素的 lens {#最原始而朴素的-lens}

最早期的 lens 技术十分朴素：将 getter 和 setter 放在一个元组中

```fsharp
type getter<'s, 'a> = 's -> 'a
type setter<'s, 'a> = 's -> 'a -> 's
type lens = getter * setter
```

当需要对一个字段进行操作时，只需要将从根目录到该字段沿途所有的这种 lens 元组，进行函数复合即可。

  这种方式虽然十分简单，但是并不高效，而且缺乏多态支持，实际使用上很受限，所以早期的 lens 技术长期没有热度，直到
van Laarhoven lens 的出现，lens 技术才变得十分强大。


## Van Laarhoven lens {#van-laarhoven-lens}

首先，van Laarhoven lens 的代码定义如下:

```fsharp
type getter<'s, 'a> = 's -> 'a
type setter<'s, 'a> = 's -> 'a -> 's
type lens = fun f s -> fmap (setter s) (f (getter s))

// 其中 f 是一个 Functor
// fmap相当于是 Functor 上的 map
// 使用 Haskell 定义则相当简洁
// lens :: (s -> a) -> (s -> a -> s) -> Lens s a
// lens getter setter = \f s -> setter s <$> f (getter s)

```

  关于 van Laarhoven lens 的大部分讲解都是围绕如何将 getter 和 setter 通过一系列操作统一在一个函数内，且转换为
上面的定义格式，但是我觉得这种方式很难理解。而反过来先看定义，再分析为什么这一个函数可以同时实现访问和写入，反而更容易理解些。

要理解 van Laarhoven lens 需要先理解两个函子(Functor):


### Functor {#functor}

[函子](https://ncatlab.org/nlab/show/functor) 是范畴上的同态，可以看作是集合上的 map 的升级版, 参考 map :

```fsharp
// map 接受一个映射函数，然后将 a 映射到 b上
map : ('a -> 'b) -> 'a -> 'b

// 函子的 fmap 也一样，只不过操作对象变成了函子
fmap : ('a -> 'b) -> F 'a -> F 'b

```


### Identity functor {#identity-functor}

[恒等函子](https://ncatlab.org/nlab/show/identity+functor) 将一个范畴上的对象和态射始终映射到它门自身，可以通过集合上的 identity function 进行理解:

```fsharp
// 恒等函数始终将任意对象映射为其自身
let identity x = x
```


### Constant functor {#constant-functor}

   [常数函子](https://ncatlab.org/nlab/show/constant+functor) 将范畴 C 上的所有对象映射到范畴 D 上的某个固定对象，且将范畴 C 上的所有态射映射为这个固定对象的恒等态射，
同样，这个函子可以通过集合上的 constant function 进行理解:

```fsharp
// 常数函数无论接受任何输入，始终都返回的是一个固定值
// 定义2更准确些，但是因为 fsharp 支持 partial application，所以两个实际上是等价的
let constant a b = a
// 或者
let mkConstant a = fun _ -> a
```


### 访问 {#访问}

有了上面的背景知识介绍，就能开始说明，访问和更新是如何实现的。

更新十分简单，将 Constant functor 带入 lens 定义即可:

```fsharp
// getConst : Const a b -> a
// getConst 相当于从 Constant 函子中取出那个固定的对象
let view lens s = getConst (lens Const s)
```

先抛除 getConst，分析下剩余部分发生了什么:

lens Const s =

fmap (setter s) (Const (getter s))

1.  getter s 能够取到需要访问的数据 x，
2.  生成常数函子 Const x
3.  根据常数函子的性质可知, fmap (setter s) (Const x) = Const x
4.  getConst (Const x) 得到 x


### 更新 {#更新}

更新的定义如下:

```fsharp
// f 是一个更新函数, f : 'a -> 'a
// f >> Identity = fun x -> Identity (f x)
let over f s = getIdentity (lens (f >> Identity) s
```

同样的，先不管 getIdentity，分析下剩余的部分:

lens (f &gt;&gt; Identity) s =

fmap (setter s) (Identity (f (getter s)))

1.  getter s 取出需要更新的旧值
2.  将旧值应用到更新函数 f 上，得到更新后的值 x
3.  生成恒等函子 Identity x
4.  fmap (setter s) (Identity x) = Identity (setter s x)
5.  setter s x 得到更新后的对象 y
6.  getIdentity (Identity y) 得到更新后的对象 y


## 实现 {#实现}

上面的所有操作在 Haskell 中实现是最简洁的，但是因为 Haskell 高度抽象的原因，反而会隐藏掉许多需要理解的细节

这里我用 fsharp 实现了一份，然后写的真的痛苦。。。许多地方不加类型标注将会得到奇奇怪怪的编译错误

```fsharp
// 注意:这里的缩进可能在导出时会发生改变，直接复制粘贴可能无法通过编译
module Lens

// fsharp 没有 Type Class，所以这里使用接口实现
type IFunctor<'a> =
    abstract fmap : ('a -> 'b) -> IFunctor<'b>

// Const 有两个构造参数，其中 b 是幻影类型
// 然后就是 Const 并不是函子, Const<'a, 'b> 类似于 Haskell 中 Const a b
// 在 Const a b 中, (Const a) 才是函子，所以这里的 fmap 里面实际上有三个类型参数
// 可以参考上面的常量函数 const a b 中，(const a) 才是真正的常量函数
type Const<'a, 'b> =
Const of 'a interface IFunctor<'b> with
    // 常量函子 (Const 'a) 在维持对 'a 的固定的情况下，将任何 'b 映射为 'c
    // 这个 fmap 中虽然看起来什么都没干, 实际上的操作发生在幻影类型上
    // fmap :: ('b -> 'c) -> Const 'a 'b -> Const 'a 'c, 更准确的说是
    // fmap :: ('b -> 'c) -> F 'b -> F 'c
    // 其中 F = Const 'a
        member this.fmap _ =
            let (Const c) = this in Const c :> IFunctor<'c>

// 恒等函子将任意 'a 映射为其自身
type Identity<'a> =
Identity of 'a interface IFunctor<'a> with
        member this.fmap (f : 'a -> 'b) =
            let (Identity id) = this in
                f id |> Identity :> IFunctor<'b>

// 辅助函数
let fmap<'a, 'b> f (F : IFunctor<'a>) = F.fmap f : IFunctor<'b>

// lens 的定义，实际没多少内容，主要是类型标注加太多，但是不加的话，编译器会推导出奇奇怪怪的结果
let inline mkLens (getter : 's -> 'a) (setter : 's -> 'b -> 'c) =
    fun (f : 'a -> IFunctor<'b>) (s : 's) -> fmap (setter s) (f (getter s)) : IFunctor<'c>

// 访问的定义，除了类型标注外，类型转换也挺烦人的
let view<'a, 's>
    (lens : ('a -> IFunctor<'a>) -> 's -> IFunctor<'s>)
    (s : 's) =
    let toFunctor = fun x -> Const x :> IFunctor<'a>
    let (Const c) = lens toFunctor s :?> Const<'a, 's> in
        c

// 更新的定义
let over<'a, 's>
    (lens : ('a -> IFunctor<'a>) -> 's -> IFunctor<'s>)
    f
    s =
    let toFunctor = fun x -> Identity x :> IFunctor<'a>
    let (Identity r) =  (lens (f >> toFunctor) s) :?> Identity<'s> in
        r
```


## 示例 {#示例}

```fsharp
open System
open Lens

// Skill 类型模拟深层级数据
type Skill = {
    Damage : int
    }

// Monster 持有一个 Skill 类型
type Monster = {
    Name : string
    Level : int
    Skill : Skill
    }

[<EntryPoint>]
let main argv =
    // Monster 上 Skill 的 Lens
    let skillLens = mkLens (fun s -> s.Skill) (fun s a -> {s with Skill = a} )

    // Skill 上 Damage 的 Lens
    let damageLens = mkLens (fun s -> s.Damage) (fun s a -> {s with Damage = a} )

    // 复合，生成从 Monster 上访问、修改 Damage 的 Lens
    let lens = damageLens >> skillLens

    let skill = {Damage = 100}
    let monster = {Name = "Monster"; Level = 14; Skill = skill}

    // 访问
    view lens monster |> printfn "Get Damage is: %O"

    // 修改
    over lens (fun _ -> 999) monster
    |> printfn "Update Damage, New Monster is: %O"

    0 // return an integer exit code
```

结果：

```bash
Get Damage is: 100
Update Damage, New Monster is: { Name = "Monster"
                                 Level = 14
                                 Skill = { Damage = 999 } }
```

  实际上成熟的 lens 库不会这么复杂的进行操作，使用起来的效果最终和命令式语言类似，而且 lens 也并不仅仅只是拿来进行数据的
访问和修改，不过其他部分需要理解的地方更多了，暂时还没时间，哈哈。
