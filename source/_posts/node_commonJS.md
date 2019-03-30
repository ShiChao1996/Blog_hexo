---
 title: node 模块机制
 tags: 
    - JavaScript
    - node
 author:
    nick: Lovae
  
 cover: /images/node_module/cover.jpg
 date: 2018-3-14
 
 subtitle:
    nodeJS 模块机制笔记
 categories: JavaScript

---
# node 模块机制

### 模块引用

示例代码如下：

```js
const fs = require("fs");
```

在 CommonJS 规范中，require 接受一个模块标志，以此引入模块的 API。

模块提供了 exports 对象来导出方法或变量，另外还有一个 module 对象，该对象即模块本身，而在 nodejs 中，文件就是模块。在 module 对象上有一个 module.exports 属性，这是其导出的内容，变量 exports 指向的地址就是 module.exports。也就是说：

```js
module.exports === exports
```

这里注意，可以在 exports 上添加属性或方法来导出，但不可修改 exports 本身的值，因为改了以后，exports 不在指向 module.exports, 也就不会被导出。如果想要导出一个类，可以：

```js
let A = {}
A.prototype.foo = foo;
...
module.exports = A;
```

### node 模块实现

> node 会将加载过的模块放入缓存，下次引用直接从缓存加载。

#### 路径分析 和 文件定位

* 核心模块，如 fs 、path、http 等，直接引用模块名。node 启动时就已经加载到内存，加载速度最快
* "." 或".."开头，相对路径查找，知道路径，查找快，但仍需动态加载，速度稍慢
* "/"开头，从根目录查找，同上
* 自定义模块，根据 `module.paths` 变量递归向上查找 node_modules 目录

#### 文件定位

> require 查找模块时，需要 fs 模块同步阻塞的判断是否存在。

require 时一般不需要指定文件后缀名，但也可以加上。如果没有后缀，node 会依次在对应路径查找 `.js`、`.json`、`.node`。如果是后两种，加上后缀名查找会稍快。

很可能最后找不到对应的`.js`、`.json`、`.node`文件，但找到的是一个目录。则会查看该目录`package.json`下main 项对应的值。示例如下:

```js
"version": "1.0.0",
  "description": "",
  "main": "webpack.config.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```

于是找到了 webpack.config.js 文件。如果没有 main 项或者不存在 package.json，则会依次查找 index.js , index.json, index.node。如果仍然没有，就按照 module.path 数组依次递归向上查找。最终找不到，则抛出异常。

#### 模块编译

node 中模块定义如下：

```js
function Module(id, parent) {
  this.id = id;
  this.parent = parent;
  this.exports = {};
  if(parent && parent.children){
    parent.children.push(this);
  }

  this.filename = null;	//定义时还不能确定该值
  this.loaded = false;
  this.children = [];
}
```

定位到具体文件后，对不同类型的文件操作不一样：

* .js 文件，通过 fs 模块同步读取后编译执行
* .node 文件，这是 c/c++ 写的扩展文件
* .json 文件，读取后通过 JSON.parse() 解析并返回结果

每一个编译后的模块都被缓存起来。

##### javaScript 模块的编译

我们前面知道有 require 方法和 exports 对象，可是这些变量和方法在哪里声明的呢？实际上，node 对读取到的 js 文件做了包装：

```js
(function (exports, require, module, __filename, __dirname) {
  // module content
});
```

node 读取 js 文件后执行的就是这个包装函数，然后得到 module.exports 。

(待续)