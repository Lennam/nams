---
title: "Js里的继承"
date: 2018-01-02T16:50:08+08:00
draft: false
---

在其他的OO语言中，最常见的两种继承方式是接口继承和实现继承。

我们知道，在JS中函数是没有签名的，函数的命名参数只是为了便利，并不是必须的，就算没有命名参数，我们还可以通过arguments获得传入函数的参数，就是这么随意~

### 原型链

我们知道每个构造函数都有一个原型对象，原型对象又有一个指向构造函数的指针，而实例内部也有一个指向原型对象的指针，如果我们让原型对象指向另一个类型的实例，这时原型对象将包含一个指向另一个原型对象的指针，而另一个原型对象又是另一个类型的实例，如此层层递进，就形成了原型链：

```javascript
function SuperType() {
  ....
}

function SubType() {
  ....
}

SubType.prototype = new SuperType();
```

当然，原型链存在一些问题，包含引用类型的原型属性会被所有实例共享，通过原型链进行继承的时候，原型会变成另一个类型的实例，于是，原先的实例属性也会变成现在的原型属性：

```javascript
function SuperType() {
  this.names = ['Nam', 'Leenam', 'Yang'];
}
function SubType() {
  
}
SubType.prototype = new SuperType();

const instance1 = new SubType();
instance1.names.push('Neil');
console.log(instance1.names);  // ["Nam", "Leenam", "Yang", "Neil"]

const instance2 = new SubType();
console.log(instance2.names);  // ["Nam", "Leenam", "Yang", "Neil"]
```

### 借用构造函数

在使用原型链的继承中，为了解决引用类型值的问题，开发人员开始使用一种叫做借用构造函数的方法，，即在子类型的构造函数中调用超类型的构造函数：

```javascript
function SuperType() {
  this.names = ['Nam', 'Leenam', 'Yang'];
}
function SubType() {
  // 借用构造函数
  SuperType.call(this)
}
SubType.prototype = new SuperType();

const instance1 = new SubType();
instance1.names.push('Neil');
console.log(instance1.names);  // ["Nam", "Leenam", "Yang", "Neil"]

const instance2 = new SubType();
console.log(instance2.names);  // ["Nam", "Leenam", "Yang"]
```

借用构造函数还可以向超类型构造函数中传递参数：

```javascript
function SuperType(name) {
  this.name = name;
}
function SubType() {
  // 借用构造函数
  SuperType.call(this, 'Nam')
}
SubType.prototype = new SuperType();

const instance1 = new SubType();
console.log(instance1.name);  // "Nam"
```

但是，使用借用构造函数方法，无法避免构造函数模式存在的问题——方法都在构造函数中定义，这样会给复用带来极大的困难。而且，超类型原型中定义的方法，在子类型中也是不可见的。

```javascript
function SuperType(name) {
  this.name = name;
}
SuperType.prototype = {
  say: function() {
    console.log(1111)
  }
}
function SubType() {
  SuperType.call(this, 'Nam')
}

const instance1 = new SubType();
instance1.say();  // TypeError: instance1.say is not a function
```

### 组合继承

如果我们把原型链和借用构造函数结合一下，我们既可以通过在原型上定义方法实现了函数复用，又能保证每个实例都有它自己的属性：

```javascript
function SuperType(name) {
  this.name = name;
  this.interest = ['sports', 'game'];
};
SuperType.prototype.say = function() {
    console.log(this.name)
};
function SubType(name, age) {
  SuperType.call(this, name);
  this.age = age
};
SubType.prototype = new SuperType();

const instance1 = new SubType('Nam', 18);
const instance2 = new SubType('Leenam', 23);

instance1.interest.push('music');
console.log(instance1.interest);  // ["sports", "game", "music"]
instance1.say();  // 'Nam'

console.log(instance2.interest);  // ["sports", "game"]
instance2.say();  // 'Leenam'
```

### 原型式继承

Douglas Crockford在2006年的一篇文章内提出了一种继承方式：

```javascript
function obj(o) {
    function F() {
        F.prototype = o;
        return new F();
    }
}
```

ES5新增的Object.create()方法规范了原型式继承，我们可以来看看它的pollyfill：

```javascript
if (typeof Object.create !== "function") {
    Object.create = function (proto, propertiesObject) {
        if (typeof proto !== 'object' && typeof proto !== 'function') {
            throw new TypeError('Object prototype may only be an Object: ' + proto);
        } else if (proto === null) {
            throw new Error("This browser's implementation of Object.create is a shim and doesn't support 'null' as the first argument.");
        }

        if (typeof propertiesObject != 'undefined') throw new Error("This browser's implementation of Object.create is a shim and doesn't support a second argument.");

        function F() {}
        F.prototype = proto;

        return new F();
    };
}
```

可以看到，核心就是Douglas Crockford提出的这种方式。

使用Object.create()来实现继承：

```javascript
function SuperType(name) {
  this.name = name;
  this.interest = ['sports', 'game'];
};
SuperType.prototype.say = function() {
    console.log(this.name)
};
function SubType(name, age) {
  SuperType.call(this, name);
};
SubType.prototype = Object.create(SuperType.prototype);
```

注意，我们使用Object.create()时子类型的原型会丢失constructor，如果我们需要constructor，可以手动添加：

```javascript
SubType.prototype.constructor = SubType;
```

如果我们需要希望继承到多个对象，可以使用混入的方式：

```javascript
SubType.prototype = Object.create(SuperType.prototype);
Object.assign(SubType.prototype, OtherSuperType.prototype);

SubType.prototype.constructor = SubType;
```