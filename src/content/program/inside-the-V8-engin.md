---
title: "JavaScript 工作原理: V8引擎内部+关于如何编写及优化代码的5条建议"
date: 2020-03-28T20:41:24+08:00
cover: "https://cdn-images-1.medium.com/max/1600/1*AKKvE3QmN_ZQmEzSj16oXg.png"
categories: ["JavaScript"]
tags: ["Nodejs", "V8", "JavaScript"]
---

## 概述

**JavaScript 引擎**是执行 JavaScript 代码的程序或解释器. 它可以作为标准的解释器或即时编译器(JIT), 以某种形式将 JavaScript 编译为字节码.

<!--more-->

这是一些实现 JavaScript 引擎比较流行的项目:

- V8: 谷歌用 C++开发的开源项目.
- Rhino: Java开发的开源项目, 由火狐基金会管理.
- SpiderMonkey: 第一个 JavaScript 引擎, 在当时用于 Netscape Navigator, 现在用于 Firefox.
- JavaScriptCore: 也叫 Nitro, 开源项目, 苹果为 Safari 开发.
- KJS: KDE的 引擎, 最初由 Harri Porten 为 KDE 项目的 Konqueror 网络浏览器开发.
- Chakra (JScript9): IE 浏览器.
- Nashorn: 由 Oracle Java 语言和工具组编写, 是 OpenJDK 的一部分.
- JerryScript: 轻量级物联网引擎.

## 为什么创建 V8 了引擎?

V8 引擎是 Google 用 C++开发的开源引擎, 被用于 Google Chrome 内部. 然而, 不像其它引擎, V8 还被用于现在流行的 Node.js 的运行时(runtime).

![](https://cdn-images-1.medium.com/max/1600/1*AKKvE3QmN_ZQmEzSj16oXg.png)

V8 最初是为了提高 web 浏览器中JavaScript 执行的性能而设计的. 为了获得速度, V8 将 JavaScript 代码转换为更为高效的机器码, 而不是使用解释器. 它通过实现**JIT(Just-In-Time) 编译器** (如 SpiderMonkey 或 Rhino(Mozilla) 等许多现代JavaScript引擎) 将 JavaScript 代码编译为机器代码. 他们主要的区别是 V8 不产生字节码或任何中间代码. 

## V8 曾经有两个编译器

在V8.5版本发布之前(今年早些时候发布), 该引擎使用了两个编译器:

- full-codegen: 一种简单且非常快速的编译器, 它产生简单且相对较慢的机器代码
- Crankshaft: 一个更复杂的 (Just-In-Time) 优化编译器, 可以生成高度优化的代码

V8 引擎还在内部使用多线程:

- 主线程正如你想的那样: 获取代码, 编译并执行.
- 还有一个单独的线程用于编译, 这样主线程可以在优化代码的同时继续执行.
- Profiler线程, 它将告诉运行时(runtime)我们在哪些方法上花费了大量时间, 以便 Crankshaft 可以优化它们.
- 还有一些线程用于扫描并进行垃圾回收.

当第一次执行代码时, V8 利用 **full-codegen** 将解析好的 JavaScript 转换成机器码, 而无需其他任何转换. 这使它可以非常快速的执行机器码. 需要注意的是, V8 不适用中间字节码的方法, 所以不需要解释器. 当你的代码运行一段时间后，Profiler 线程已经收集了足够的数据以确定哪种方法应该进行优化.

接下来, **Crankshaft**在另一个线程里进行优化. 它将 JavaScript 抽象语法树转换成称为 **Hydrogen** 的高级静态单赋值(SSA)表示, 并尝试去优化 **Hydrogen** 图. 大多数的优化都是发生在这一级别的.

## 内联(Inlining)

>The first optimization is inlining as much code as possible in advance. Inlining is the process of replacing a call site (the line of code where the function is called) with the body of the called function. This simple step allows following optimizations to be more meaningful

第一个优化是预先内联尽可能多的代码. 内联是用被调用函数的主体替换函数调用点(即调用函数的代码行)的过程. 这个简单的步骤以下的优化更有意义.

![](https://cdn-images-1.medium.com/max/1600/0*RRgTDdRfLGEhuR7U.png)

### 隐藏类

JavaScript 是一门基于原型的语言: 没有使用克隆来创建类和对象的过程. JavaScript 也是一门动态语言, 它可以在实例化后很容易地从对象中添加或删除属性. 

大多数 JavaScript 解释器都使用类似字典结构(基于哈希函数)在内存中存储对象的属性值. 这种结构使得 JavaScript 检索属性值的的时候比其它非动态编程语言(如 Java, C 或 C#) 在计算上代价更昂贵. 在 Java 中, 所有对象的属性都是在编译之前由固定的模型决定的, 不能在运行时添加或删除(C#的动态类型不在这讨论). 因此, 这些属性(或指向这些属性的指针)可以作为连续的缓存区存储在内存中, 每个缓存区具有固定大小的偏移量. 偏移量的大小可以很容易根据属性值的类型确定, 这在 JavaScript 中是不可能的, 因为属性值的类型在运行时可能会更改. 

由于使用字典查找对象属性在内存中的位置非常低效, 因此 V8 使用了一种不同的方法: 隐藏类. 隐藏类的工作方式和 Java 等语言中使用的固定对象模型是相似的, 只是它是在运行时创建的. 现在, 让我们看看它实际的样子:

```js
function Point(x, y) {
    this.x = x;
    this.y = y;
}
var p1 = new Point(1, 2);
```

一旦 `new Point(1, 2)` 被调用, V8 将创建一个隐藏的类 `C0`.

![](https://cdn-images-1.medium.com/max/1600/1*pVnIrMZiB9iAz5sW28AixA.png)

`C0` 中没有定义属性, 所以它是空的.

一旦执行了第一个语句 `this.x = x`（在Point函数中), V8 将基于 `C0` 创建第二个隐藏类 `C1`. `C1` 描述了属性 `x` 可以在内存中被找到的位置(相对于对象的指针). 在这种情况下, `x` 存储在偏移量为 0 的位置, 这意味着当将存储器中的 `Point` 对象看做连续缓冲器时，第一个偏移将对应于属性 `x`. V8 也会使用类转换来更新 `C0`, 如果将一个属性 `x` 添加到 `Point` 对象, 则隐藏类会从 `C0` 切换到 `C1`. 下面的 `Point` 对象当前的隐藏类是 `C1`.

![](https://cdn-images-1.medium.com/max/1600/1*QsVUE3snZD9abYXccg6Sgw.png)

每次向对象添加新的的属性时, 旧的隐藏类都会通过转换路径被转换到新的隐藏类. 隐藏类的转换很重要, 因为它们允许以相同方式创建的对象来共享隐藏类. 如果两个对象共享一个隐藏类, 并且将相同的属性添加到这两个对象, 则转换过程将确保这两个对象接收相同的新隐藏类及其附带的所有优化代码. 

当执行 `this.y = y` (在 `Point` 函数内部, 执行完后 `this.x = x` 后)时, 会重复此过程.

一个名为 `C2` 的新隐藏类被创建, 将类转换添加到 `C1`, 声明如果属性 `y` 添加到点对象(已包含属性 `x` )，则隐藏类应更改为 `C2`，并且点对象的隐藏类将更新为 `C2`.

![](https://cdn-images-1.medium.com/max/1600/1*spJ8v7GWivxZZzTAzqVPtA.png)

隐藏类的转换取决于属性被添加到对象的顺序. 看下面的代码片段: 

```js
function Point(x, y) {
    this.x = x;
    this.y = y;
}
var p1 = new Point(1, 2);
p1.a = 5;
p1.b = 6;
var p2 = new Point(3, 4);
p2.b = 7;
p2.a = 8;
```

现在，你可以假设对于 `p1` 和 `p2`, 将使用相同的隐藏类和转换. 实际上并不相同, 对于 `p1`, 首先将添加属性 `a`, 然后添加属性 `b`. 但是, 对于 `p2`, 首先分配 `b`, 然后再分配 `a`. 因此, `p1` 和 `p2` 由于转换路径的不同, 最终会有不同的隐藏类. **在这种情况下，最好以相同的顺序初始化动态属性，以便可以重用隐藏的类.** 

### 内联缓存(Inline Caching)

V8 利用了另一种优化动态类型语言的技术, 称为内联缓存. 内联缓存依赖于这样的情况: 重复调用同一对象的上同一方法. 更加深入的解释可以在[这里](https://github.com/sq/JSIL/wiki/Optimizing-dynamic-JavaScript-with-inline-caches)找到.

我们将简要说明内联缓存的一般概念(如果没有时间去看上面的深入解释).

那么它是如何工作的呢? V8 维护了一个最近调用的方法中作为参数传递的对象的类型缓存, 并使用该信息对将来可能传递的参数类型做出预判. 如果V8能够很好地预判将要传递给方法的对象的类型, 它可以绕过计算访问对象属性的过程, 而是使用以前查找中存储的信息来访问对象的隐藏类. 

那么隐藏类和内联缓存的概念是如何相关的呢? 每当在特定对象上调用方法时, V8 引擎必须对该对象的隐藏类进行查找, 以便确定访问特定属性的偏移量. 在同一个方法对同一个隐藏类成功调用两次之后, V8 省略了隐藏类查找, 只需将属性的偏移量添加到对象指针本身. 对于该方法将来的所有调用, V8引擎假定隐藏类没有改变, 并使用以前查找中存储的偏移量直接跳转到特定属性的内存地址. 这大大提高了执行速度.

内联缓存也是为什么同一类型的对象共享隐藏类的重要的原因. 如果创建两个具有相同类型和不同隐藏类的对象(如前面的示例所示), V8 将无法使用内联缓存, 因为即使这两个对象的类型相同, 它们对应的隐藏类也会为它们的属性分配不同的偏移量. 

![](https://cdn-images-1.medium.com/max/1600/1*iHfI6MQ-YKQvWvo51J-P0w.png)

## 编译成机器码

一旦 Hydrogen 图被优化, Crankshaft 就会将其降低到称为 Lithium 的较低级别表示. 大多数 Lithium 的实现都是针对架构的. 寄存器分配发生在这个级别.

最后, Lithium 被编译为机器码. 然后发生了另一件事，叫做OSR: 堆栈替换. 在我们开始编译和优化一个运行时间很长的方法之前, 我们很可能在运行它. V8 不会忘记它刚刚缓慢执行的内容, 以便重新开始优化版本. 相反, 它将转换我们拥有的所有上下文(堆栈、寄存器), 以便在执行过程中切换到优化版本. 这是一项非常复杂的任务, 要记住, 在其他优化中, V8最初已经内联了代码. V8 不是唯一能够做到这一点的引擎. 

有一种称为去优化的保护措施, 可以进行相反的转换, 并在引擎所做的假设不再成立的情况下恢复到非优化代码.

## 垃圾回收

对于垃圾回收, V8 采用传统的标记--清除的扫描方法处理过期代码 (old generation). 标记阶段应该停止执行 JavaScript. 为了控制 GC 成本并使执行更加稳定, V8 使用增量标记, 而不是遍历整个堆, 它试图标记每个可能的对象, 它只遍历一部分堆, 然后恢复正常的代码执行. 下一次 GC 将继续从之前的遍历停止的位置开始. 这允许在正常执行期间非常短的暂停. 如前所述, 扫描阶段由单独的线程处理. 

## Ignition 和 TurboFan

随着2017年早些时候发布 V8 5.9, 引入了新的执行流程. 这个新的管道在实际的 JavaScript 应用程序中实现了更大的性能改进和显着的内存节省.

新的执行流程建立在 [Ignition](https://github.com/v8/v8/wiki/Interpreter)，V8 的解释器和 [TurboFan](https://github.com/v8/v8/wiki/TurboFan), V8 的最新优化编译器之上. 

您可以查看 V8 团队关于[此主题](https://v8project.blogspot.bg/2017/05/launching-ignition-and-turbofan.html)的博客帖子.

由于 V8 5.9 版本的出炉, V8 将不再使用 `full-codegen` 和 `Crankshaft` (自2010 年起服务于 V8 的技术),  因为V8团队努力跟上新的 JavaScript 语言功能，这些功能需要优化.

这意味着整体 V8 将会有更简单和更易维护的架构.

![](https://cdn-images-1.medium.com/max/1600/0*pohqKvj9psTPRlOv.png)

这些改进只是一开始. 新的 Ignition 和 TurboFan 管道为进一步优化铺平了道路, 这将在未来几年内提升 JavaScript 性能并缩小 V8 在 Chrome 和 Node.js 中的占用空间.

最后, 这里有一些关于如何编写优化的, 更好的 JavaScript 的提示和技巧. 您可以很容易地从上面的内容中得到这些, 但是, 为了方便起见, 这里有一个摘要:

## 如何编写优化的 JavaScript
1. **对象属性的顺序**:始终以相同的顺序实例化对象属性, 以便可以共享隐藏类和随后优化的代码.
2. **动态属性**: 在实例化后向对象添加属性将强制隐藏类更改, 并任何为先前隐藏类优化的方法变慢. 所以, 使用在构造函数中分配对象的所有属性来代替.
3. **方法**: 重复执行相同方法的代码将比只执行一次的代码(由于内联缓存)运行得快.
4. **数组**: 避免键不是增量数字的稀疏数组. 稀疏数组是一个哈希表. 这种阵列中的元素访问消耗较高.  另外, 尽量避免预分配大型数组, 最好按需分配, 自动增加. 最后, 不要删除数组中的元素, 它使键稀疏.
5. **标记值**: V8 使用 32 位表示对象和数字. 它使用一个位来标记它是一个对象(flag = 1)还是一个称为 SMI(SMall Integer)的整数(flag = 0), 因为它是31位. 因此，如果一个数值大于  31位，V8 将会把数字转换为 double 类型, 并创建一个新对象把数字放在里面. 尽可能使用  31位有符号数字, 以避免将数字转换为 JS 对象的昂贵操作.

## 资源
- https://docs.google.com/document/u/1/d/1hOaE7vbwdLLXWj3C8hTnnkpE0qSa2P--dtDvwXXEeD0/pub
- https://github.com/thlorenz/v8-perf
- http://code.google.com/p/v8/wiki/UsingGit
- http://mrale.ph/v8/resources.html
- https://www.youtube.com/watch?v=UJPdhx5zTaw
- https://www.youtube.com/watch?v=hWhMKalEicY


> 原文: [How JavaScript works: inside the V8 engine + 5 tips on how to write optimized code](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)    
> 作者: [Alexander Zlatkov](https://blog.sessionstack.com/@zlatkov)
