---
title: "Js的基本类型"
date: 2017-5-17T11:50:48+08:00
draft: false
---

Javascript的内置类型总共有7种：

- Undefined
- Null
- String
- Number
- Boolean
- Symbol（ES6新增）
- Object

其中除了Object为引用类型外，其他都为基本类型。

我们可以使用 ***typeof*** 操作符来检测基本类型：

```javascript
typeof undefined === 'undefined';  // true
typeof 'Nam' === 'string';  // true
typeof 1106 === 'number';  // true
typeof true === 'boolean';  //  true
typeof Symbol() === 'symbol'; // true

//  当我们使用typeof操作符对null进行验证的时候
typeof null; // ‘object',  What?
```

可以看到，我们typeof null返回的是object类型，这是一个特殊情况，正确的返回应该是null，但是这个bug由来已久，在javascript中存在了近20年，如果对它进行修复，会牵扯到太多web系统，将会产生更多的bug，所以这个问题也许永远得不到修复了…

如果我们想检测null的类型，可以使用复合条件：

```javascript
const a = null;
(!a && typeof a === 'object);  // true
```

在javascript中，变量是没有类型的，只有值才有。

当使用typeof对变量进行检测的时候，检测的是变量持有的值：

```javascript
let a = 1106;
typeof a;  // 'number'

a = 'Nam';
typeof a;  // 'string'
```

### Undefined

Undefiend类型只有一个值，即为undefined。

当我们声明一个变量，但未赋值时，它的值便是undefined：

```javascript
let a;
typeof a;  // 'undefined'
```

Undefined类型的类型转换：

```javascript
Boolean(undefined);  // false
Number(undefined);  // NaN
String(undefined);  // 'undefined'
```

### Null

Null类型也只有一个值null，从逻辑上看，它表示一个空对象的指针，如果定义的变量将来用于保存对象，可以将该变量初始化为null。

Null类型的类型转换：

```javascript
Boolean(null);  // false
Number(null);  // 0
String(null);  // 'null'
```

### String

在Javascript中的String类型的值是不可变的，不可变指的是字符串的成员函数不会改变其原始值，而是创建并返回一个新的字符串：

```javascript
let a = 'nam';
b = a.toUpperCase();
a === b;  // false
a;  // 'nam'
b;  // 'NAM'
```

因为String类型的值不可变，我们可以借用数组的非变更方法来处理字符串：

```javascript
const c = Array.prototype.join.call(a, '@');
c;  // 'n@a@m'
```

String类型的类型转换：

```javascript
//  当字符串中含有非数字时，String类型转换为Number类型会得到NaN
//  空字符串会得到0
Number('nam');  // NaN
Number('nam1106');  // NaN
Number('1106');  // 1106
Number('');  // 0

Boolean('');  // false
Boolean('nam');  // true

```

### Number

Number类型是基于[IEEE754](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)标准来实现的，该标准通常也被称为“浮点数”，Javascript使用的是64位二进制格式(二进制浮点数)。所以Javascript没有真正意义上的整数.

特别大和特别小的数字默认用指数格式显示，与toExponential()方法的输出结果相同：

```javascript
const a =6E10;
a;  // 60000000000
a.toExponential();  // '6e+10'

const b = a * a; //
b;  //  3.6e + 21

```

二进制浮点数有一个最大的问题：

```javascript
0.1 + 0.2 === 0.3； // false

```

因为二进制浮点数中的0.1和0.2并不是十分精确，它们相加的结果并非刚好等于0.3，所以会返回false。

如果我们要判进行类似0.1 + 0.2是否与0.3相等的判断，我们可以设置一个误差范围，从ES6开始，这个范围定义在Number.EPSILON中：

```javascript
const a = 0.1 + 0.2；
const b = 0.3;
Math.abs(a -b) < Number.EPSILON;  // true

```

要检测一个值事否是整数，可以使用ES6中的Number.isInteger()方法：

```javascript
Number.isInteger(1106);  // true
Number.isInteger(1106.000);  // true
Number.isInteger(11.06);  // false

```

Number类型中有一个特殊的值NaN，它与任何值都不想等，包括它自己，任何涉及到NaN的操作都会返回NaN。

使用isNaN（）方法可以检测一个值是不是NaN。

Number类型的类型转换：

```javascript
String(1106);  // '1106'
String(NaN);  // 'NaN'
Boolean(1106):  // true
Boolean(0);  // false
```

### Boolean

Boolean类型只有两个值：true和false

往Boolean（）方法中传入空数组或者空对象时也会返回true

### Symbol

ES6新引入了一种新的基本类型Symbol，它表示独一无二的值。

Symbol值通过Symbol（）方法生成：

```javascript
let a = Symbol();
typeof a;  // 'symbol'
```

Symbol（）方法接受一个字符串作为参数，该字符串只是表示对当前Symbol的描述，因此相同参数的Symbol（）返回值并不相等：

```javascript
let a = Symbol();
let b = Symbol()
a === b;  // false

let c = Symbol('Nam');
let d = Symbol('Nam');
c === d;  // false
```