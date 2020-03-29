---
title: "让我们揭开 JavaScript 关键字 new 的神秘面纱"
date: 2020-03-29T00:59:54+08:00
lastmod: 2020-03-29T00:59:54+08:00
cover: "https://cdn-images-1.medium.com/max/2000/1*yF1csYj3s_p4lBZW8L0iCA.jpeg"
categories: ["JavaScript"]
tags: ["JavaScript"]
---


![](https://cdn-images-1.medium.com/max/2000/1*yF1csYj3s_p4lBZW8L0iCA.jpeg)
      
周末, 我完成了 Will Sentance 的[《JavaScript: The Hard Parts》](https://frontendmasters.com/courses/javascript-hard-parts/)课程. 听起来也许这并不是度过周末最光荣的方式, 但是实际上我觉得完成这门课很轻松, 也很有趣. 它涉及到了函数式编程, 高阶函数, 闭包以及异步 JavaScript的内容.    

<!--more-->

对我来说, 这个课程的亮点是他如何将 JavaScript 的方法扩展到了面向对象编程(OOP), 并揭开了 new 运算符的背后的魔法. 我现在已经全面了解了使用 `new` 运算符时内部发生了什么. 

## JavaScript 的面向对象编程
![](https://cdn-images-1.medium.com/max/1600/0*SjgGIxtpI-X9CnNl.)

面向对象编程是一种基于"对象"概念的编程范式.数据和函数(属性和方法)捆绑在一个对象中.
    
JavaScript 中的对象是键值对的集合. 这些键值对就是对象的属性. 属性可以是数组, 函数, 对象本身, 或者任何基本数据类型, 比如字符串或数字.    
    
我们的 JavaScript 工具箱中有哪些用于创建对象的技术呢?
    
假设我们正在为我们刚设计的游戏创建用户. 我们将如何存储用户的信息, 例如他们的名字, 点数以及实现诸如点数递增的工具方法呢? 以下是创建基本对象的方法.

### 方法1: 对象字面量
```js
let user1 = {
  name: "Taylor",
  points: 5,
  increment: function() {
    user1.points++;
  }
};
```
JavaScript 对象字面量是一个用大括号包裹的键值对的列表. 上面的例子中, 我们创建了`user1`对象, 并在其中存储了相关的数据.
    
### 方法2: Object.create()
`Object.create(proto, [ propertiesObject ])`    
`Object.create` 方法接受两个参数:
1. proto: 要创建的对象的原型. 它必须是一个对象或者 `null`.
2. propertiesObject: 新对象的属性. 这个参数是可选的.
    
基本上, `Object.create` 将返回一个继承自你传入对象的新对象.
```js
let user2 = Object.create(null);
user2.name = "Cam";
user2.points = 8;
user2.increment = function() {
  user2.points++;
}
```
上面的基本对象创建选项是重复的, 每一个都需要手动去创建.    
我们怎么解决这个问题呢?

## 解决方案
### 方案1: 使用函数生成对象
一个简单的解决方案是编写一个函数来创建新用户.
```js
function createUser(name, points) {
  let newUser = {};
  newUser.name = name;
  newUser.points = points;
  newUser.increment = function() {
    newUser.points++;
  };
  return newUser;
}
```
你现在可以在函数的参数中输入信息来创建用户.
```js
let user1 = createUser("Bob", 5);
user1.increment();
```
但是, 上面示例中的 `increment` 函数只是对原始 `increment` 函数的复制. 这不是编写代码的好方法, 如果这个函数有所改动, 你需要对每个对象进行更改.

### 方案2: 使用 JavaScript 的原型特性
与 Python 和 Java 等面向对象的语言不通, JavaScript 没有真正意义上的类. 它使用原型和原型链的概念进行继承.    

当你创建了一个数组, 你自然而然的就可以访问内置方法, 比如 `Array.join`, `Array.sort` 和 `Array.filter` 等. 这是由于数组对象继承了 `Array.prototype` 的属性.    
     
![prototype](https://cdn-images-1.medium.com/max/1600/1*9CKiUI1JDrcJPWfqYdBjFg.png)
    
每个 JavaScript 函数都有一个默认为空的原型属性. 你可以给这个原型属性添加函数, 这种形式的函数我们称之为方法. 当执行这个继承函数时, 它的 `this` 指向继承的对象.
```js
function createUser(name, points) {
  let newUser = Object.create(userFunction);
  newUser.name = name;
  newUser.points = points;
  return newUser;
}
let userFunction = {
  increment: function() {this.points++};
  login: function() {console.log("Please login.")};
}
let user1 = createUser("Bob", 5);
user1.increment();
``` 

当创建 `user1` 对象时, 一个绑定了 `userFunction` 的原型链就形成了.    
在堆栈中调用 `user1.increment()` 时, 解释器会在全局的内存中查找 `user1`, 然后再查找 `increment` 函数, 但是不会找到它. 所以解释器会查找原型链上的下一个对象, 并在那里找到 `increment` 函数.    
    
### 方案3: new 和 this 关键字

![new and this](https://cdn-images-1.medium.com/max/1600/1*mF32LKAQAr5BrbyON_i6bg.jpeg)

**new 运算符用于创建具有构造函数的对象的实例.**

当我们用 `new` 来调用构造函数时, 会自动执行以下操作:
- 创建一个新对象
- 把 `this` 绑定到新对象
- 构造函数的原型对象成为新对象的 `__proto__` 属性
- 返回新对象

这太棒了, 因为它是自动的, 减少了重复的代码!
```js
function User(name, points) {
 this.name = name; 
 this.points = points;
}
User.prototype.increment = function(){
 this.points++;
}
User.prototype.login = function() {
 console.log(“Please login.”)
}
let user1 = new User(“Dylan”, 6);
user1.increment();
```

使用原型模式, 每个方法和属性都会直接添加到对象的原型上.   

解释器将在原型链上向上查找, 并找到位于 `User` 上的 `increment` 函数, 该函数本身也是一个包含信息的对象. 记住 ---- JavaScript 中的所有函数也是对象. 现在解释器已经找到了它所需要的东西, 它可以创建一个新的本地执行上下文来运行 `user1.increment`.

#### 附注: `__proto__` 与 `prototype` 之间的区别 
如果你已经对 `__proto__` 和 `prototype` 感到困惑, 不要担心! 你不是唯一一个对此感到困惑的人.    

`prototype` 是构造函数的属性, 它决定了构造对象上的 `__proto__` 属性.
因此, `__proto__` 是创建的一个引用, 它就是我们熟知的原型链联系.

### 方案4: ES6 语法糖    

![class](https://cdn-images-1.medium.com/max/1600/1*WXIGjN5Dnwv70NcRvVWfyg.jpeg)

其他允许我们在对象"构造函数"中编写公共方法.ECMAScript6 引入了 `class` 关键字, 它允许我们编写类似其他语言的普通类. 实际上, 它是 JavaScript 原型行为的语法糖.    
```js
class User {
  constructor(name, points) {
    this.name = name;
    this.points = points;
  }
  increment () {
    this.points++;
  }
  login () {
    console.log("Please login.")
  }
}
let user1 = new User("John", 12);
user1.increment();
```
在方案3, 中使用的 `User.prototype.functionName` 来精确的实行了关联的方法. 在这个方案中, 我们得到了同样的结果, 但是语法看起来更那个清晰.

## 结论
现在我们已经了解了更多关于 JavaScript 中创建对象的不同方法. 虽然声明 `class` 和 `new` 运算符相对易于使用, 但了解内部的运行机制非常重要.     

回顾一下, 当用 `new` 调用构造函数时, 会自动执行以下操作:
- 创建一个新对象
- 把 `this` 绑定到新对象
- 构造函数的原型对象成为新对象的 `__proto__` 属性
- 返回新对象


> 原文: [Let’s demystify JavaScript's 'new' keyword](https://medium.freecodecamp.org/demystifying-javascripts-new-keyword-874df126184c)    
> 作者: [Cynthia Lee](https://medium.freecodecamp.org/@cynthialixinlee)  
> 发布时间: 2018-04-24 

