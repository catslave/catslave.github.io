---
layout: post
title: Thinking in TypeScript
category: Language
description: 什么时候要用TypeScript？它解决了JavaScript的什么问题？
---

## 1. TypeScript简介

### TypeScript是什么？
TypeScript是由微软开发的一门静态类型、弱类型的编程语言。它是JavaScript的超集，也就是基于JavaScript扩展的语言。

### 开发它的初衷
由于JS在代码编写上的灵活性，导致在运行时会带来很多不必要的类型错误。因此，TS为JavaScript添加了类型系统，它能够对代码进行静态类型检查。这样能够很大程度的减少代码在运行时的类型错误。

### TypeScript和JavaScript的区别
TS完全兼容JavaScript，你能够用TS运行现有的任何JavaScript代码。TS代码编译后，会转换成原生的JavaScript代码，TS不会改变JavaScript的任何行为。它只是在编译阶段新增了类型检查，并且在编译结束后擦除这些类型。

## 2. 安装TypeScript

### 安装TS

我们可以通过NPM来安装typescript。

全局环境下安装typescript

```npm install -g typescript ```

tsc是typescript的命令行工具，我们可以通过tsc来编译TS代码。

### 编写TS代码

TypeScript文件名后缀是.ts结尾

### 编译TS文件

使用tsc能够将TS代码编译成JS文件

### 运行TS代码

TS本质仍是JS，所以最后执行的仍然是JS代码。我们通过TSC编译器，将TS代码编译成JS代码后，可以通过node执行JS文件。

我们也可以通过ts-node插件来直接运行TS文件，它本质也是先将TS转换成JS，然后执行JS。

## 3. 类型系统

TypeScript 是一个基于 JavaScript 的静态类型检查器，能够在编译期间检查代码错误，特别是针对值类型错误的检查，能够给出错误提示。

TypeScript 在 JavaScript 基于上添加了一层 - 类型系统（这里用图表示更直观）。


### 3.1 静态类型检查

#### 类型检查

TS是增加了类型系统的JS，为JS添加了静态类型检查。在编译阶段除了能够检查到代码的类型错误，还能够推断出可能的类型。支持TS的IDE，更能够在代码编写阶段进行自动类型检查和智能提示。

#### 类型推导

根据值类型和值结构类型进行类型推导和类型匹配。对于值结构类型来说，只要结构的子集满足了目标结构类型，TS也认为它们两个类型匹配。

显示指定 Type Annotations：默认情况下，TS会隐式的自动根据上下文推导出值类型，我们也可以显示的给变量声明类型，来指定值的类型。

类型断言 Type Asserations：对于返回的值，你也可以使用 as 语法来指定具体的类型。类型断言和类型注解一样，在编译阶段会被擦除。类型断言 as 在实践中使用的也比较多，特别对于调用第三方函数或者接收回调的时候，收到到一个any入参，可以通过as来自动转换成我们自定义的类型。

#### 类型系统

这一套的类型注解、类型检查、类型推导就是TS为JS扩展的类型系统。它能够在代码编译阶段对值类型进行自动推导和匹配，并检查类型错误。在编译完成后，又能够将类型擦除，生成原生的JS代码。通过TS的类型检查，能够很大程度上减少JS的运行时错误。

## 4. 基础类型

### Primitive 原始类型

JavaScript支持的Primitive：string, number, bigint, boolean, undefined, symbol, null。 TypeScript的此基础上新增了void。

### Array 数组

JS中的数组可以存储任意数据类型，通过new Array()来声明一个数组。在TS中，我们可以通过原始类型来定义数组，并且数组的类型是指定的。因为TS具有类型系统，所以不允许这样的事情发生，一个数组里的数据类型只能说唯一的。number[], string[] 来声明不同数据类型的数组。

### any 类型

如果你不想对值进行任何的类型声明，那么可以使用any类型，any能够匹配任意类型。你如果没有对值类型进行声明，TS也无法通过上下文进行推断，那么会默认为any类型。

一个变量或者参数如果没有声明类型，TS隐式的认为是 any 类型。

但在代码里使用any类型不是一个好的规范，因为any类型跳过了类型检查，也就失去了类型检查的意义，你应该要尽可能的知道值的类型。我们可以通过设置编译选项noImplicitAny禁用any的默认推断。

### Functions 函数

我们可以通过Type Annotation来指定函数入参和返回值的类型。对于匿名函数，TS会自动进行上下文类型推导。

我们可以使用“?”来声明函数入参的可选，例如，function f(x? number) {}，实际上，参数x会被声明为 number | undefined 的联合类型。如果不传递参数，x默认为undefined。

TS函数如果没有声明返回值，也没有返回任何值，那么函数调用默认的结果是void。但是，在JS中函数如果没有返回值，默认是unfined。这里需要注意，在TS中void和undefined不是同一个类型。

Parameter Destructuring 参数解构：对于拆解参数对象非常方便，可以参数声明一个对象类型。这样在调用函数的时候，能够自动对参数进行拆解成定义的对象。实战中常常用到的一个小技巧。

### Union Type 联合类型

TS允许你通过“|”操作符来合并多种类型，这种类型称为联合类型。也就是说，你可以给一个值声明多种类型，使用“|”来连接不同类型。例如，function printId(id: number | string) {} 函数参数id你既可以传递数字，也可以传递字符串。

通过联合类型，TS可以非常容易的进行值类型的匹配，只要满足其中任意一种类型即可。但是在使用联合类型的值时要注意，我们需要对值先做类型检查，这样TS才能推断当前你要使用的类型是哪一个。

### Interface 接口

使用interface关键字可以定义一个对象类型。
、
与此相似功能的是Type Alias 类型别名，类型别名可以定义任何类型。

Interfaces和Type Alias的区别是，Interface 变量名可以重复，也就是扩展合并，而Type Alias必须唯一。

### Enums 枚举类型

Enums 是TS为JS新增的特性，用于声明类型可能属于一个集合中的一种，枚举不是类型系统的一部分。

### Narrow 缩小类型

将联合类型缩小为一种类型，也就是在代码中处理每一种类型。在JS中，常常使用if条件和typeof来处理每种类型的逻辑。

JS前置知识：typeof 用于获取Primitives类型，instanceOf 作用于对象类型

### Object Type 对象类型

如果一个函数有多个入参，我们一般会将这些参数封装成一个对象进行传递。对象的包装我们可以直接使用对象字面量类型或者声明一个interface类型。

定义一个对象的属性，我们一般需要考虑三个元素：属性的类型，属性是否可选（属性名后面紧跟“?”），以及属性是否只读（属性名前声明 readonly）。

Index Signatures：有时候你知道接收到的值是什么类型，但是不知道值的名称，那么可以使用索引签名。例如，key/value形式的值。这在实战中，是一个编码小技巧。

## 5. 复杂类型

### Decorators

新增特性，支持meta programming。但在TS 5.0之前个功能只是一个实验性的功能，需要显示开启才能支持（experimentalDecorators选项：true）。

Decorator本质是一个函数，通过function实现。在运行时被调用。

Decorator可以装饰在Class、Method、Property、Parameter上面。且可以装饰多个Decorators，它们的执行顺序是从上到下，返回结果是从内到外，类似递归调用。

Decorator的入参和返回值，以及执行顺序需要理清楚。

### Mixin

除了通过继承类的方式来复用组件，我们还可以通过合并部位类来复用组件，这就是Mixin方式。

### Modules

## 6. 工程配置管理

### tsconfig.js

tsconfig.json 是 TS 工程的配置文件，tsc 命令就是根据 tsconfig.json 的定义，来编译工程文件输出 js 文件的。tsc 是一个编译器，能够将 ts 文件编译成 js。

### CompilerOptions

--target 指定TS需要编译成的JS版本，例如，es5、es6/es2015等等。也就是说，如果你在TS中使用了es6的语法，例如，常量const和箭头函数=>等，那么TS会自动将其转换成es5语法。但是TS无法将所有es6的语法都转换成es5，比如，Promise语法。这种情况下，就需要使用另一个编译参数 --lib。

--lib 将指定的library库添加到运行时，默认继承--target参数，即target指定es6，那么lib默认使用的是es6的库。--lib更大的用处在于，你可以添加不属于当前es版本的库，例如，在es5的target中添加es6的新特性，甚至es2017、es2018、es2019的新特性。


## 7. 其它思考

### 其它类型检查语言

Flow 也是一个 JavaScript 静态类型检查工具，它不是一个新语言，它只是一个 annotation 注解。它直接作用于 JavaScript 代码，使用 Babel 编译器会对添加了注解的 JS 代码进行类型检测。

### coding 感受/思维转变

有服务端/强类型语言编程经验的人，上手 TS 会非常的熟悉，因为它的写法与大多数强类型语言类似，还包括了 class、interface，等概念。

