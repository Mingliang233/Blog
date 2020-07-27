---
title: 什么是函数式编程？
---

函数式编程（以下简称FP）是一种编程范式，是一种软件构建的方式。与之相关的有面向对象编程和面向过程编程。

FP更倾向于代码的简洁，可预测，易测试。以下是FP的特性：

#### 纯函数的组合

什么是纯函数（pure functions）？

纯函数的特点是：

给定同样的输入，总是获得同样的输出，并且没有副作用（ side-effects）



#### 避免shared state

什么是shared state？

shared state可以是全局作用域或者闭包作用域中的变量。

这样的shared state带来了一些问题。

* 检查代码不方便。比如，当我们要想弄清楚一个function的作用时，由于收到

shared state的影响，我们就要额外了解和这个shared state相关的一些操作。

* 可能引发一些条件竞争问题。函数调用的顺序变化可能影响执行结果。

  

#### Immutability不可变性

不可变性是FP中强调的一个中心思想。不可变即一个对象一旦被声明就不能再更改。



#### 避免side-effects

任何引起了外部变化的函数都产生了side-effects

比如：

- Modifying any external variable or object property (e.g., a global variable, or a variable in the parent function scope chain)
- Logging to the console
- Writing to the screen
- Writing to a file
- Writing to the network
- Triggering any external process
- Calling any other functions with side-effects

#### Reusability Through Higher Order Functions

函数通过**higher order function**可以实现重用

A **higher order function** is any function which takes a function as an argument, returns a function, or both. Higher order functions are often used to:

- Abstract or isolate actions, effects, or async flow control using callback functions, promises, monads, etc…
- Create utilities which can act on a wide variety of data types
- Partially apply a function to its arguments or create a curried function for the purpose of reuse or function composition
- Take a list of functions and return some composition of those input functions

#### Declarative vs Imperative

FP都是声明式的范式，意味着不会明确表现flow control

Imperative是命令式，描述flow control：怎么做

declarative是声明式，描述data flow：做什么

下面是imperative:

```javascript
const doubleMap = numbers => {
  const doubled = [];
  for (let i = 0; i < numbers.length; i++) {
    doubled.push(numbers[i] * 2);
  }
  return doubled;
};

console.log(doubleMap([2, 3, 4])); // [4, 6, 8]
```

下面是declarative：

```javascript
const doubleMap = numbers => numbers.map(n => n * 2);

console.log(doubleMap([2, 3, 4])); // [4, 6, 8]
```

#### 总结

函数式编程倾向以下几点：

- Pure functions 而不是 shared state 和side effects
- Immutability over mutable data  不可变
- Function composition over imperative flow control 组合函数而不是 imperative flow control
- Lots of generic, reusable utilities that use higher order functions to act on many data types instead of methods that only operate on their colocated data
- Declarative rather than imperative code (what to do, rather than how to do it)
- Expressions over statements
- Containers & higher order functions over ad-hoc polymorphism



##### 参考文献

精通javascript--什么是函数式编程<https://medium.com/javascript-scene/master-the-javascript-interview-what-is-functional-programming-7f218c68b3a0>