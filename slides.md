---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
highlighter: shiki
---

# CommonJS modules in Node.js

<a href="https://github.com/xbrave/slides.git" target="_blank" alt="GitHub"
  class="abs-br m-6 text-xl icon-btn opacity-50 !border-none !hover:text-white">
  <carbon-logo-github />
</a>

---
 
# 为什么要引入模块化
先问为什么，再问是不是

- 变量和方法不容易维护，容易污染全局作用域
- 依赖的环境主观逻辑偏重，代码较多就会比较复杂
- 大型项目资源难以维护，特别是多人合作的情况下
- 复用性较差，多人合作容易相互干扰

---

# `require()`做了什么
参考[这段伪代码](https://nodejs.org/api/modules.html#modules_all_together)，简单概括如下:
1. 优先加载诸如`path`,`fs`等核心模块
2. 先从缓存中找对应的模块是否已加载，没有则进行下一步
3. 加载对应的以`/`, `./`, `../`开头的文件，如果不带文件后缀，则按照`.js`, `.json`,`.node`依次尝试，有的话加载对应的文件内容
4. 加载对应的以`/`, `./`, `../`开头的文件夹，先查找文件夹下的`package.json`，如果`package.json`定义了"main",则加载"main"对应的内容，没有则寻找对应的文件夹下的index文件，然后按照第三步的加载文件顺序依次查找并加载
5. 加载对应的`node_modules`文件夹下的内容
6. 抛出"not found"错误

Read More: 
* [Webpack Module Resolution](https://webpack.js.org/concepts/module-resolution/)
* [enhanced-resolve](https://github.com/webpack/enhanced-resolve)

---

# 缓存
可以从[`require.cache`](https://nodejs.org/dist/latest-v14.x/docs/api/modules.html#modules_require_cache)获取已缓存的相关模块，对应的每个缓存模块的数据结构类似于:
```js {2|4|15|16}
  {
    id: '/xbrave/debug/foo.js',
    path: '/xbrave/debug',
    exports: {},
    parent: Module {
      id: '.',
      path: '/xbrave/debug',
      exports: {},
      parent: null,
      filename: '/xbrave/debug/index.js',
      loaded: false,
      children: [Array],
      paths: [Array]
    },
    filename: '/xbrave/debug/foo.js',
    loaded: true,
  }
``` 
---

# MyOwnModule Class
参考缓存中所描述的module结构和[源码](https://github.com/nodejs/node/blob/910efc2d9a69ac32205d64e4835df481fc057f02/lib/internal/modules/cjs/loader.js#L168)我们先定义一个MyOwnModule类:
```js
const path = require('path');

function MyOwnModule(id = '') {
  this.id = id;
  this.path = path.dirname(id);
  this.exports = {};
  this.filename = null;
  this.loaded = false;
}
```
这里省略了源码里parent和children相关的属性

---

# MyOwnModule.prototype.require()
这里主要做了如下几件事:
1. 初始化一个缓存对象
2. 调用内部的`_loader()`方法
3. 通过 `_resolveFilename()`方法获取相应的文件绝对路径
4. 然后从缓存中找模块是否已加载，加载了直接返回对应的缓存内容，否则新生成一个Module实例，再返回对应的加载结果
```js {1|3|6|8-11|all}
MyOwnModule._cache = Object.create(null);
MyOwnModule.prototype.require = function(id) {
  return MyOwnModule._load(id);
}
MyOwnModule._load = function(request) {
  const filename = MyOwnModule._resolveFilename(request);
  const cachedModule = MyOwnModule._cache[filename];
  if (cachedModule !== undefined) return cachedModule.exports;
  const module = new MyOwnModule(filename);
  MyOwnModule._cache[filename] = module;
  module.load(filename);
  return module.exports;
}
```
---

# MyOwnModule._resolveFilename()
`_resolveFilename`的方法在源码中比较复杂，这里我们简单实现一下，主要做的工作和之前说到加载顺序一致:
1. 获取文件的resolve路径，并检测有没有文件后缀
2. 如果有后缀直接返回，没有的按照`_extensions`定义的顺序依次寻找，找到的直接返回对应的路径
```js {2-3|8|all}
MyOwnModule._resolveFilename = function(request) {
  const filename = path.resolve(request);
  const ext = path.extname(filename);
  if (!ext) {
    const extensions = Object.keys(MyOwnModule._extensions);
    for (let i = 0; i <= extensions; i++) {
      const currentFile = `${filename}${extensions[i]}`;
      if (fs.existsSync(currentFile)) return currentFile;
    }
  }
  return filename;
}
```
---

# MyOwnModule.prototype.load()
`load()`方法主要是根据对应的文件后缀依次处理文件内容，然后将当前Module的内部属性`loaded`置为true。
```js
MyOwnModule.prototype.load = function(filename) {
  const extname = path.extname(filename);
  MyOwnModule._extensions[extname](this,filename);
  this.loaded = true;
}
```
我们平常开发中一般涉及到的是`.js`和`.json`的文件后缀，我们分别对两个文件后缀的内容进行处理

---

# [vm.runInThisContext(code[, options])](https://nodejs.org/api/vm.html#vm_vm_runinthiscontext_code_options)
### 官方释义:
vm.runInThisContext() compiles code, runs it within the context of the current global and returns the result. Running code does not have access to local scope, but does have access to the current global object.
### 官方demo:
```js
const vm = require('vm');
let localVar = 'initial value';

const vmResult = vm.runInThisContext('localVar = "vm";');
console.log(`vmResult: '${vmResult}', localVar: '${localVar}'`);

const evalResult = eval('localVar = "eval";');
console.log(`evalResult: '${evalResult}', localVar: '${localVar}'`);
```
---

# `.js`文件后缀
对`.js`文件后缀的处理,主要是调用了[`vm.runInThisContext`]()方法将拼接好的字符串代码转换为实际代码，并在最外层包裹了一层，将对应的五个变量注入了模块内部。相关代码如下:
```js {3|8|13-17|19|all}
MyOwnModule._extensions = Object.create(null);
MyOwnModule._extensions['.js'] = function(module, filename) {
  const fileContent = fs.readFileSync(filename, 'utf-8');
  module._compile(fileContent, filename);
} 

MyOwnModule.wrapper = [
  '(function (exports, require, module, __filename, __dirname) { ', '\n});',
];

MyOwnModule.prototype._compile = function(content, filename) {
  const wrappedContent = MyOwnModule.wrapper[0] + content + MyOwnModule.wrapper[1];
  const compiler = vm.runInThisContext(wrappedContent, {
    filename,
    lineOffset: 0,
    displayErrors: true
  });
  const dirname = path.dirname(filename);
  compiler.call(this.exports, this.exports, this.require, this, filename, dirname);
}
```
---

# `.json`文件后缀
对`.json`文件后缀的文件处理就比较简单了，读取文件内容，然后调用`JSON.parse()`方法返回处理之后的结果:
```js {2|all}
MyOwnModule._extensions['.json'] = function(module, filename) {
  const content = fs.readFileSync(filename, 'utf-8');
  module.exports = JSON.parse(content);
}
```
---

## 总结
1. `Node.js`的`CommonJS module`并没有什么特别的地方，主要难点在于对于`.js`文件的处理方式和`vm.runInThisContext`的理解
2. 每个模块对应的`exports`,`require`,`module`, `__filename`, `__dirname`都不是全局变量，而是模块加载的时候注入的

<br />
<br />
<br />

## 参考
1. [Module:CommonJS modules official document](https://nodejs.org/api/modules.html)
2. [Node.js loader.js Source Code](https://github.com/nodejs/node/blob/910efc2d9a69ac32205d64e4835df481fc057f02/lib/internal/modules/cjs/loader.js)