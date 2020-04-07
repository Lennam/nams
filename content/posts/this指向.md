---
title: "This指向"
date: 2017-10-22T15:15:09+08:00
draft: false
---

刚学JS时，总是对this的指向摸不着头脑，所以对它的使用，总是小心翼翼，能避就避，但在一些场景中，不使用this，会让代码变得丑陋。比如：

```javascript
function sayHi() {
    return 'Hello, I'm ' + this.name;
}
const me = {
    name: 'Nam'
}
sayHi.call(me);  //  Hello,I'm Nam

//  而当我们不使用this时，必须要显示的传入一个上下文对象

function sayHi(obj) {
    return 'Hello, I'm ' + obj.name;
}
const me = {
    name: 'Nam'
}
sayHi(me);  //  Hello,I'm Nam
```

显然，this提供了一种更优雅的方式来隐式的传递一个对象。当我们使用的模式越来越复杂，显示的传递上下文对象会让代码变得越来越混乱，使用this就可以很好的解决这个问题。

 

### this的指向

关于this的指向，总有一个原则：this总是指向调用调用函数的那个对象。

我们来看具体的栗子：

#### 1.独立函数调用

```javascript
var a = 1;
function say() {
    console.log(this.a)
}
say();  // 1

//  当我们使用严格模式的时候
var a = 1;
function say() {
    ‘use strict';
    console.log(this.a)
}
say();  // TypeError: this is undefined
```

#### 2.new函数调用

```javascript
function Person(name) {
     this.name = name;
}
var man = new Person('Nam');
man.name;  // 'Nam'
```

#### 3.call(),apply()和bind()

```javascript
//  通过call()在调用say()时强制绑定this到指定对象
//  这种绑定方法并不是永久的，只在调用时暂时绑定
//  apply()绑定this的机制和call()一样，他们的区别是在其他参数上
function say() {
     console.log(this.name)
}
var person = {
     name: 'Nam'
}
say.call(person);  // 'Nam'

//  通过bind()强制绑定
function someoneDo(something) {
    return this.name + ' is '  + something;
}
var person = {
    name: 'Nam'
}
var namDo = someoneDo.bind(person);
console.log(namDo('flying'));  // 'Nam is flying'
```

#### 4.上下文对象函数调用

```javascript
function say() {
  console.log(this.name);
}
var person = {
  name: 'Nam',
  say: say
}
person.say();  // 'Nam'
```

当一个对象引用另一个对象时：

```javascript
function say() {
  console.log(this.name);
}
var person = {
  name: 'Nam',
  say: say
}
var anotherPerson = {
  name: 'Leenam',
  person: person
}
anotherPerson.person.say();  // 'Nam'

// 显然只有对象引用的最后一层会影响调用位置
```

这里有一个丢失this绑定的问题：

```javascript
function say() {
  console.log(this.name);
}
var person = {
  name: 'Nam',
  say: say
}
var anotherSay = person.say;
var name = 'window';
anotherSay();  // window
```

可以看到，这里anotherSay的this绑定到了全局对象。因为anotherSay对person.say的引用，实际上是对say本身的引用，因此anotherSay本身是一个独立函数调用，因此this绑定到了全局对象上。

还有一种问题发生在传入回调函数时：

```javascript
function say() {
  console.log(this.name);
}
function doSomething(fn) {
  fn();
}
var person = {
  name: 'Nam',
  say: say
}
var name = 'window'
doSomething(person.say)；  // 'window
```

参数传递实际上也是一种隐式赋值，所以结果和上一个栗子一样。

### this绑定优先级

我们已经清除上述四种情况中的this绑定，那如果在某种情境下上述四种情况都适用，我们应该已哪种为准呢？

毫无疑问，独立函数调用的优先级是最低的。

call、applay、bind和上下文对象函数调用呢？

```javascript
function say() {
  console.log(this.name);
}
var person = {
  name: 'Nam',
  say: say
}
var anotherPerson = {
  name: 'Leenam',
  say: say
}
person.say();  // 'Nam'
person.say.call(anotherPerson);  // 'Leenam'

```

显然，call、apply、bind的优先级更高。

现在让我们来对比一下new函数调用和call、apply、bind：

```javascript
function person(name) {
  this.name = name;
}
var man1 = {};
var setName = person.bind(man1);
setName('Nam');
console.log(man1.name);  // 'Nam'

var man2 = new setName('Leenam');
console.log(man1.name);  // 'Nam'
console.log(man2.name);  // 'Leenam'

```

setName被绑定到了man1上，但是new setName并没有把man1的name改为Leenam。很奇怪对不对，关于[bind](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)我们可以查看它的pollyfill去了解它是怎么实现的。

 

最后，我们可以依照下面的顺序来判断this：

1. 函数是否在new中调用？如果是的话，this指向新创建的对象
2. 是否通过call、apply、bind进行绑定？如果是的话，this指向绑定的对象
3. 是否在某个上下文对象中调用？如果是的话，this指向上下文对象
4. 如果以上都不是，非严格模式下this绑定到全局对象，严格模式下绑定到undefined