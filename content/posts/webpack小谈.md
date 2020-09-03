---
title: "Webpack小谈"
date: 2019-08-31T11:05:57+08:00
draft: false
---

在当今的前端开发链中，打包工具已经成为了不可获取的一环，它帮我们把现代Js应用的各种静态模块构建生成依赖图，并根据依赖图生成一个或多个浏览器能识别的bundle。

它大致的流程如下：

```javascript
入口文件  ------->  依赖图（dependency graph）  ------->  bundle
```

这其中有两个关键点：

1. 依赖图的生成：

   依赖图（dependency graph）是包含了应用中所有使用到的模块，它表示了各个模块之间的依赖关系，webpack会根据入口文件，层层递归找出所有的依赖，使得webpack可以获取非代码资源，如 images 或 web 字体等，并把他们提供给应用程序。

   ![image](/webpack/WX20200831-160455@2x.png)

   

2. 根据依赖图产出bundle：

   webpack在内部生成依赖图后，会根据依赖图遍历并读取所有的文件，把代码资源根据顺序合并为一个或多个bundle，而图片和字体等则会放入指定的文件夹以供应用程序读取。

   ![image](/webpack/WX20200831-160455@2x.png)

所以webpack整个流程可以表示为：

![image](/webpack/WX20200831-161855@2x.png)

------

我们要了解webpack的内部原理，可以从上面的两个关键点入手。设想一个简单的应用：

![image](/webpack/WX20200831-163434.png)

有一个入口文件index.js，其代码如下：

```javascript
import play from './player.js';

play();
```

player.js和song.js分别如下：

```javascript
// player.js
import { SONG_NAME } from './song.js';

const play = () => {
  console.log(SONG_NAME)
};

export default play;


// song.js
export const SONG_NAME = 'With or Without You';
```

我们新建一个index.html文件，并引入index.js:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Music Player</title>
</head>
<body>

<script src="./src/index.js"></script>

</body>
</html>
```

然后我们在浏览器中打开index.html，发现浏览器报错：

![image](/webpack/WX20200831-164400@2x.png)

显然，我们引入的是一个js文件，而浏览器无法识别js文件中的**import**语法，所以我们要设法把应用中的所有模块集中到一起并转化为浏览器能识别的js代码。涉及到代码的转换，熟悉编译原理的同学就知道接下来就该抽象语法树(Abstract Syntax Tree，*AST*)出场了。

关于AST，感兴趣的同学可以自己下去研究，我在这里引用维基的解释:

> 在计算机科学中，**抽象语法树**（**A**bstract **S**yntax **T**ree，AST），或简称**语法树**（Syntax tree），是**源代码语法结构**的一种抽象表示。它以**树状**的形式表现**编程语言**的语法结构，树上的每个节点都表示源代码中的一种结构。之所以说语法是“抽象”的，是因为这里的语法并不会表示出真实语法中出现的每个细节。

回想之前webpack的两个关键点，首先是生成依赖图。要生成依赖图，我们就必须找出一个模块中所有的导入语句，在这之前，我们先看看index.js转化成AST后是怎样的（使用[astexplorer](https://astexplorer.net/)，因截图过长，把代码定位属性隐藏）：

![image](/webpack/WX20200901-102351.png)

index.js转化为AST后，在body里有几个声明：**ImportDeclaration**,**EmptyStatement**(表示代码中的空行，可以忽略),**ExpressionStatement**。我们从ImportDeclaration中可以看到一条import语句被分成了不同的标识节点，我们可以遍历所有的ImportDeclaration及其节点，便可以搜集到一个模块中的所有依赖。

接下来我们新建一个bundler.js，去实现上面所说的部分：

```javascript
// bundler.js
// 为了读取文件和转化成ast我们需要fs、path和@babel/parser模块，而遍历ast我们需要@babel-traverse模块
const fs = require('fs')
const path = require('path')
const parser = require('@babel/parser')
const traverse = require('@babel/traverse').default

// 获取模块信息
const createModuleInfo = (file) => {
    const body = fs.readFileSync(file,'utf-8')
    const ast = parser.parse(body,{
      sourceType:'module' //表示我们要解析的是ES模块
    });
  
  	const dependencies = []
    // 遍历AST搜集所有的导入信息
    traverse(ast, {
        ImportDeclaration({node}) {
            dependencies.push(node.source.value)
        }
    })
  
    console.log(dependencies)
}

createModuleInfo('./src/index.js')
```

执行以上代码，我们便得到了index.js的依赖：

```json
[ './player.js' ]
```

当然，出了入口文件index.js的依赖，其它模块也有它自己的依赖，为了区分不同模块间的依赖，我们可以增加一些额外的信息：

```javascript
// 修改createModuleInfo函数，新增返回信息，
const createModuleInfo = (filename) => {
    const body = fs.readFileSync(filename,'utf-8')
    const ast = parser.parse(body,{
      sourceType:'module' //表示我们要解析的是ES模块
    });
  
  	const dependencies = []
    // 遍历AST搜集所有的导入信息
    traverse(ast, {
        ImportDeclaration({node}) {
            dependencies.push(node.source.value)
        }
    })
  
 	 	const id = ID++;
  
  	return {
        id,
        filename,
        dependencies
    }
}
```

此时，当我们调用createModuleInfo方法，便会得到下面的信息：

```json
{ id: 0,
  filename: './src/index.js',
  dependencies: [ './player.js' ] }
```

接下来，就可以开始着手构建依赖图了。我们可以从入口文件开始，递归每一个文件的dependencies：

```javascript
// 在bundler.js中新增createGraph方法
const createGraph = (entry) => {
    const entryInfo = createModuleInfo(entry)
    const queue = [entryInfo]
    for (const file of queue) {
        const dirname = path.dirname(file.filename)
        file.dependencies.forEach(dep => {
            const relativePath = dep; // 相对路径
            const absolutePath = path.join(dirname, relativePath); // 绝对路径
            const child = createModuleInfo(absolutePath); // 通过绝对路径找到模块
            queue.push(child);
        })
    }
    console.log(queue)
    return queue
}
```

执行后我们可以得到一个依赖列表，这就是我们的依赖图，之后我们将根据依赖图去生成最终的bundle：

```json
[ { id: 0,
    filename: './src/index.js',
    dependencies: [ './player.js' ] },
  { id: 1,
    filename: 'src/player.js',
    dependencies: [ './song.js' ] },
  { id: 2, filename: 'src/song.js', dependencies: [] } ]
```

------

要生成最终的bundle，我们只需要把依赖图里所有文件的代码集中到一起就行，并让其能在浏览器中运行即可。把代码集中到一起简单，但是要让其在浏览器中运行还有两个问题需要解决：**目前各个模块的代码使用es6写的，其无法直接在浏览器中运行，我们还需要使用其它工具把它转换成es5代码。**显然，我们这时候需要babel把es6的代码转换成为es5的代码：

```javascript
....
// 新增babel的引入
const babel = require('@babel/core')

const createModuleInfo = (filename) => {
    const body = fs.readFileSync(filename,'utf-8')
    const ast = parser.parse(body,{
      sourceType:'module' //表示我们要解析的是ES模块
    });
  
  	const dependencies = []
    // 遍历AST搜集所有的导入信息
    traverse(ast, {
        ImportDeclaration({node}) {
            dependencies.push(node.source.value)
        }
    })

  	// 遍历的时候使用bable把es6的代码转化为es5，并赋予新的code属性
    // 这里使用babel的transformFromAst方法把es6转换成es5，感兴趣的同学可以去
  	// bable官网查看详细介绍
    const {code} = babel.transformFromAst(ast, null, {
        presets: ['@babel/preset-env'],
    });

    const id = ID++;
  
    return {
        id,
        filename,
        dependencies,
        code
    }
}
```

运行后我们可以看到，在依赖图中多了code字段，并且相应的代码已经转换成es5：

```json
[{
        id: 0,
        filename: './src/index.js',
        dependencies: ['./player.js'],
        code: '"use strict";\n\nvar _player = _interopRequireDefault(require("./player.js"));\n\nfunction _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }\n\n;\n(0, _player["default"])();'
    },
    {
        id: 1,
        filename: 'src/player.js',
        dependencies: ['./song.js'],
        code: '"use strict";\n\nObject.defineProperty(exports, "__esModule", {\n  value: true\n});\nexports["default"] = void 0;\n\nvar _song = require("./song.js");\n\nvar play = function play() {\n  console.log(_song.SONG_NAME);\n};\n\nvar _default = play;\nexports["default"] = _default;'
    },
    {
        id: 2,
        filename: 'src/song.js',
        dependencies: [],
        code: '"use strict";\n\nObject.defineProperty(exports, "__esModule", {\n  value: true\n});\nexports.SONG_NAME = void 0;\nvar SONG_NAME = \'With or Without You\';\nexports.SONG_NAME = SONG_NAME;'
    }
]    
```

为了生成bundle的时候更加方便，我们可以在依赖图中新增一个mapping字段，来映射每个依赖的id：

```javascript
const createGraph = (entry) => {
    const entryInfo = createModuleInfo(entry)
    const queue = [entryInfo]
    for (const file of queue) {
        const dirname = path.dirname(file.filename)
        file.mapping = {} // 新增mapping
        file.dependencies.forEach(dep => {
            const relativePath = dep; // 相对路径
            const absolutePath = path.join(dirname, relativePath); // 绝对路径
            const child = createModuleInfo(absolutePath); // 注意我们得通过绝对路径找到模块
            file.mapping[relativePath] = child.id; // 把依赖的id存进mapping
            queue.push(child);
        })
    }

    return queue
}
```

执行后输出：

```json
[{
        id: 0,
        filename: './src/index.js',
        dependencies: ['./player.js'],
        code: '"use strict";\n\nvar _player = _interopRequireDefault(require("./player.js"));\n\nfunction _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }\n\n;\n(0, _player["default"])();',
        mapping: {
            './player.js': 1
        }
    },
    {
        id: 1,
        filename: 'src/player.js',
        dependencies: ['./song.js'],
        code: '"use strict";\n\nObject.defineProperty(exports, "__esModule", {\n  value: true\n});\nexports["default"] = void 0;\n\nvar _song = require("./song.js");\n\nvar play = function play() {\n  console.log(_song.SONG_NAME);\n};\n\nvar _default = play;\nexports["default"] = _default;',
        mapping: {
            './song.js': 2
        }
    },
    {
        id: 2,
        filename: 'src/song.js',
        dependencies: [],
        code: '"use strict";\n\nObject.defineProperty(exports, "__esModule", {\n  value: true\n});\nexports.SONG_NAME = void 0;\nvar SONG_NAME = \'With or Without You\';\nexports.SONG_NAME = SONG_NAME;',
        mapping: {}
    }
]
```

ok，接下来我们开始把所有模块代码合并成一个bundle，我们新建一个bundle函数：

```javascript
// 浏览器引入bundle代码要能执行，所以把代码包裹进一个立即执行函数
const bundle = (graph) => {
  const result = `
    (function(modules) {
        ...
    })({
        ${graph}
    })`;
}
```

注意bable转为es5后的代码，有一个require和exports需要我们自己去定义。

想一下require要做什么，其实只是把模块里的代码执行一下，所以我们把模块里的代码都用一个函数包裹起来，然后在require函数里直接调用被包裹起来的模块代码，并对对传入立即执行函数的数据做一些处理，再在require里面写一个localRequire把require传入模块：

```javascript
const bundle = (graph) => {
  let modules = '';
    graph.forEach(module => {
        modules += `
            ${module.id}: [
                function(require, module, exports) {
                    ${module.code}
                },
                ${JSON.stringify(module.mapping)}
            ],
        `;
    });
  const result = `
    (function(modules) {
        function require(id) {
            const [fn, mapping] = modules[id]; // 得到function 和依赖 
            function localRequire(relativePath) {
                return require(mapping[relativePath]);
            }
            const module = {
                exports: {}
            };
            fn(localRequire, module, module.exports);
            return module.exports;
        }
    })({
        ${modules}
    })`;
}
```

最后，把result写入bundle.js：

```javascript
fs.mkdirSync('./dist');
fs.writeFileSync('./dist/bundle.js',result)
```

然后在index.html里面引入：

```html
<script src="./dist/bundle.js"></script>
```

这样我们就能在浏览器里看到运行结果啦：

![image](/webpack/WX20200903-145038.png)

