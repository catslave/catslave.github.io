---
layout: post
title: Thinking in TypeScript
category: Language
description: 什么时候要用TypeScript？它解决了JavaScript的什么问题？
---

## TypeScript

对 TypeScript 进行简介

例如，它是一门强类型语言，基于 ES 规范实现，并覆盖了 JavaScript 的所有语法，可以编译成 JavaScript。

也就是说，TS 也是运行在 JS 引擎上的，需要有 JS 运行时环境，例如，Node.js 或者

那它比 JS 多了什么？

### 强类型

也就是类型定义，我们知道 JS 一门动态语言，也就是说变量类型可以在运行时指定，这样在 coding 的时候非常灵活，不会带来太多的约束。但是编码的灵活性会带来一定的风险，比如，函数调用的返回值类型不对，会导致运行出错。？还有其它原因吗？

TS 在 ES 基础之上，新增了类型定义规范。通过强类型约束，能够让代码更健壮，？但是否会缺失掉一些灵活行呢？这些灵活性体现在哪里？如果一个返回值返回的值类型不是预期的，程序要能正常执行？

### 专属语法糖

哪些语法糖是 TS 提供的？

1、解构：从一个数据对象中提取出另一个对象，剔除不需要的属性。

省去一个个属性赋值。

2、as：将数据自动转换成另一个对象。

类型断言：https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions
省去了一个个属性赋值。

### 还有其它特点吗？

### TS 是如何执行的

TS 是编译成 JS 后再执行的，所以本质来说，TS 只有编译器，就是将 TS 编译成 JS，然后通过 JS 正常方式进行运行。ts-node 工具可以直接运行 TS，即这个运行时可以直接允许 TS，内部是先将 TS 转换成 JS，然后用 node.js 进行执行。省去了一般的需要先将 TS 编译成 JS，再通过 node.js 执行 js。也就是说，通过 ts-node 命令可以执行执行 ts 文件，例如：
{% highlight typescript %}
ts-node index.ts
{% endhighlight %}

当然，本质上还是在执行 JS 文件。

tsconfig.json 是 TS 工程的配置文件，tsc 命令就是根据 tsconfig.json 的定义，来编译工程文件输出 js 文件的。tsc 是一个编译器，能够将 ts 文件编译成 js。

### 其它框架中集成 TS

### coding 感受/思维转变

有服务端/强类型语言编程经验的人，上手 TS 会非常的熟悉，因为它的写法与大多数强类型语言类似，还包括了 class、interface，等概念。

### 集成 TS 会扼杀灵活性吗？

比如，现在前端的成熟框架，是否可以集成 TS？有了 TS 真的能带来更好的代码质量吗？
