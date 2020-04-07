---
title: "How to use Proxy"
date: 2018-02-03T14:31:29+08:00
draft: false
---

今天看到一篇关于Proxy的文章，挺有意思，所以翻译一下。

原文地址：[How to use JavaScript Proxies for Fun and Profit](https://medium.com/dailyjs/how-to-use-javascript-proxies-for-fun-and-profit-365579d4a9f8)

以下是翻译：

Javascript最近有一个新鲜的但未得到广泛使用的特性：[JavaScript proxies](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy).

使用Javascript proxies你可以包装一个现有的对象，并拦截对其属性或者方法的访问，即使这些属性或方法并不存在！

### Hello World Proxy

让我们从一个基本的‘hello world’例子开始吧：

```javascript
const wrap = obj => {
  return new Proxy(obj, {
    get(target, propKey) {
        console.log(`Reading property "${propKey}"`)
        return target[propKey]
    }
  })
}
const object = { message: 'hello world' }
const wrapped = wrap(object)
console.log(wrapped.message)
```

这段代码输出：

```javascript
Reading property "message"
hello world
```

在这个例子中，我们只是访问对象的属性和方法前做了一些事。然后我们返回了原本的属性和方法。

你也可以用一个set函数来拦截对属性的修改。

这可能可以用于验证属性或类似的东西，但我认为这个新特性很有前景，我希望一些已Proxy为核心功能的框架出现。所以我一直在思考这些问题，然后有了一些创意：

#### 一个20行代码关于API的SDK

如上所述，你可以拦截一个方法的调用…甚至这个方法不存在。当一个被Proxy代理的对象上的方法被调用时，get函数会被调用并返回一个动态生成的函数。**如果不需要，你完全可以不去触碰被代理的对象**。

根据这个思维，你可以解析被调用的方法，并在运行的时候动态实现它的功能！举个栗子，当我们使用api.getusers()调用的时候，可以用代理在API中创建一个GET/users。通过这个约定我们可以更进一步，让api.postItems({name: ‘Item name’})使用第一个参数作为请求主体来调用POST/items。

让我们来看看完整的实现：

```javascript
const { METHODS } = require('http')
const api = new Proxy({},
  {
    get(target, propKey) {
      const method = METHODS.find(method => 
        propKey.startsWith(method.toLowerCase()))
      if (!method) return
      const path =
        '/' +
        propKey
          .substring(method.length)
          .replace(/([a-z])([A-Z])/g, '$1/$2')
          .replace(/\$/g, '/$/')
          .toLowerCase()
      return (...args) => {
        const finalPath = path.replace(/\$/g, () => args.shift())
        const queryOrBody = args.shift() || {}
        // You could use fetch here
        // return fetch(finalPath, { method, body: queryOrBody })
        console.log(method, finalPath, queryOrBody)
      }
    }
  }
)
// GET /
api.get()
// GET /users
api.getUsers()
// GET /users/1234/likes
api.getUsers$Likes('1234')
// GET /users/1234/likes?page=2
api.getUsers$Likes('1234', { page: 2 })
// POST /items with body
api.postItems({ name: 'Item name' })
// api.foobar is not a function
api.foobar()
```

这个栗子中我们代理了一个空对象{}，因为所有的方法都是动态生成的，我们并不需要去实际的包装一个具有方法的对象。

如果你不喜欢这样，也可以通过不同的方式实现。

#### 使用更易懂的方式进行数据结构查询

如果你有一个关于people的数组，你可以这样去查询：

```javascript
arr.findWhereNameEquals('Lily')
arr.findWhereSkillsIncludes('javascript')
arr.findWhereSkillsIsEmpty()
arr.findWhereAgeIsGreaterThan(40)
```

proxy让我们为所欲为！我们可以声明一个proxy去包装一个数组，解析方法调用让我们可以像上面的栗子一样查询。

我实现了一小部分：

```javascript
const camelcase = require('camelcase')
const prefix = 'findWhere'
const assertions = {
  Equals: (object, value) => object === value,
  IsNull: (object, value) => object === null,
  IsUndefined: (object, value) => object === undefined,
  IsEmpty: (object, value) => object.length === 0,
  Includes: (object, value) => object.includes(value),
  IsLowerThan: (object, value) => object === value,
  IsGreaterThan: (object, value) => object === value
}
const assertionNames = Object.keys(assertions)
const wrap = arr => {
  return new Proxy(arr, {
    get(target, propKey) {
      if (propKey in target) return target[propKey]
      const assertionName = assertionNames.find(assertion =>
        propKey.endsWith(assertion))
      if (propKey.startsWith(prefix)) {
        const field = camelcase(
          propKey.substring(prefix.length,
            propKey.length - assertionName.length)
        )
        const assertion = assertions[assertionName]
        return value => {
          return target.find(item => assertion(item[field], value))
        }
      }
    }
  })
}
const arr = wrap([
  { name: 'John', age: 23, skills: ['mongodb'] },
  { name: 'Lily', age: 21, skills: ['redis'] },
  { name: 'Iris', age: 43, skills: ['python', 'javascript'] }
])
console.log(arr.findWhereNameEquals('Lily')) // finds Lily
console.log(arr.findWhereSkillsIncludes('javascript')) // finds Iris
```

这就像[expect](https://github.com/mjackson/expect)使用proxy编写断言库那样。

另一个想法是创建一个使用API去查询数据库：

```javascript
const id = await db.insertUserReturningId(userInfo)
// Runs an INSERT INTO user ... RETURNING id
```

#### 检测async functions

因为你有了拦截方法调用的能力，所以当一个方法调用返回了一个promise，在promise完成时你也可以追踪它。根据这个想法，我又弄了一个栗子去监视一个对象的异步方法，并显示一些统计信息。

你有一个类似于下面这个栗子的服务，当一个方法调用时你能包装它：

```javascript
const service = {
  callService() {
    return new Promise(resolve =>
      setTimeout(resolve, Math.random() * 50 + 50))
  }
}
const monitoredService = monitor(service)
monitoredService.callService() // we want to monitor this
```

完整的栗子：

```javascript
const logUpdate = require('log-update')
const asciichart = require('asciichart')
const chalk = require('chalk')
const Measured = require('measured')
const timer = new Measured.Timer()
const history = new Array(120)
history.fill(0)
const monitor = obj => {
  return new Proxy(obj, {
    get(target, propKey) {
      const origMethod = target[propKey]
      if (!origMethod) return
      return (...args) => {
        const stopwatch = timer.start()
        const result = origMethod.apply(this, args)
        return result.then(out => {
          const n = stopwatch.end()
          history.shift()
          history.push(n)
          return out
        })
      }
    }
  })
}
const service = {
  callService() {
    return new Promise(resolve =>
      setTimeout(resolve, Math.random() * 50 + 50))
  }
}
const monitoredService = monitor(service)
setInterval(() => {
  monitoredService.callService()
    .then(() => {
      const fields = ['min', 'max', 'sum', 'variance',
        'mean', 'count', 'median']
      const histogram = timer.toJSON().histogram
      const lines = [
        '',
        ...fields.map(field =>
          chalk.cyan(field) + ': ' +
          (histogram[field] || 0).toFixed(2))
      ]
      logUpdate(asciichart.plot(history, { height: 10 })
        + lines.join('\n'))
    })
    .catch(err => console.error(err))
}, 100)
```

------

Javascript的proxy真是强大无比。

它增加了一点额外的开销，却让我们可以在运行时通过方法的名字动态的实现它们，这让我们的代码更加优雅和易读。目前我还没做过任何的基准测试，入股哦你想在实际的生产中使用它们，最好先做一下性能测试。

然而在开发中使用它们是没有问题的，例如为了更好的调试去监控异步函数。