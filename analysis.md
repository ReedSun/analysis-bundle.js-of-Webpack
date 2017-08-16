# 分析webpack打包后的文件

## 前言

作为前端，总是在工作中使用 Webpack 作为模块打包工具，却从来也没有分析过 Webpack 打包后的文件是什么样子的。这可不行，今天我就写了一个小例子，来分析一下 Webpack 生成的 bundle.js 到底是什么。

- Webpack 版本：3.4.1

- 代码地址：@TODO

## 文件结构

```javascript
// index.js
import a from './a'
import {b1, b2} from './b'

console.log(`a: ${a.name}`)
console.log(`b1: ${b1.name}; b2: ${b2.name}`)
```

```javascript
// a.js
export default {
	name: 'a'
}
```

```javascript
// b.js
import c from './c'

console.log(`c: ${c.name}`)

export let b1 = {
  name: 'b1'
}
export let b2 = {
	name: 'b2'
}
export let b3 = {
	name: 'b3'
}
```

```javascript
// c.js
export default {
	name: 'c'
}
```

##　生成文件

刚才的代码通过 Webpack 打包，生成了一个122行的 JS 文件，代码太长啦就不放在文章中了，大家可以在[这里](@TODO)查看。

## 分析

不难看出，实际上 bundle.js 是一个立即执行函数，其参数 `modules` 为一个数组，包含了我们所有要打包的模块。

这个立即执行函数主要分为4个部分：

- [第一部分](@TODO)，定义一个对象 `installModlues` 来保存 Webpack 已注册的模块。

- [第二部分](@TODO)，定义一个函数 `__webpack_require__` 来实现的模块的加载。这里是 Webpack 管理模块的核心。

- [第三部分](@TODO)，在 `__webpack_require__` 这个函数上绑定一些属性。

- [第四部分](@TODO)，调用`__webpack_require__`函数，开始加载模块。

### `modules` 参数分析

立即执行函数的参数 `modules` 是一个数组，其中数组中的每一项为一个将模块包裹起来的立即执行函数。并且将模块中的 `import语句`，`export语句`进行了转换。我们拿一个模块来分析, 看看他具体是什么做的：

```javascript
/* 2 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {
"use strict";
/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "a", function() { return b1; });
/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "b", function() { return b2; });
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__c__ = __webpack_require__(3);

console.log(`c: ${__WEBPACK_IMPORTED_MODULE_0__c__["a" /* default */].name}`)

let b1 = {
  name: 'b1'
}
let b2 = {
	name: 'b2'
}

/***/ }),
```

嗯，这个是一个立即执行函数，在这个立即执行函数定义了三个参数，这三个参数都是在 `__webpack_require__` 中传入的。

import 导入比较简单，将 import 替换为 `__webpack_require__` 就可以导入模块。（详见[`__webpack_require__`函数分析](@TODO)）

export 导出，则是调用 `__webpack_require__.d` 方法（ d 为 define 的简写），将输出的变量或方法绑定到模块（最终会保存到 `installModules[modulesId].export` 中）中 。（详见 [`__webpack_require__` 中的绑定的属性和方法分析](@TODO)）

注意，我们在[源代码](@TODO)中还导出了一个并没有被其他模块导入的 `b3` 属性，但是这个属性并没有出现在[打包后的文件]()中，这就是 webpack 的特性 [Tree Shaking](https://webpack.js.org/guides/tree-shaking/) 的功劳了，这个特性会把我们导出但没使用的部分不打包进 `bundle` 中，从而精简代码。

### `__webpack_require__` 函数分析

这个函数是 Webpack 管理模块的核心。它接收一个参数 `moduleId` 被引入模块的 Id ，来确定要引入的模块。而模块 Id 实际上就是就是 `module` 这个数组的索引。

这个函数主要分为如下几个部分：

- [第一部分](@TODO)，判断 `installModules` 中是否已经注册过这个模块（`installModules` 的属性 `moduleId` 是否存在），如果注册过，直接返回已注册模块的 `exports` 属性（模块的输出）。

- [第二部分](@TODO)，如果没有注册过，则定义一个对象`module`（注意区别刚才立即执行函数参数 `modules`），并将其注册到 `installModules` （绑定到 `installModules` 的属性 `moduleId` 中），这个对象包含以下三个属性：

	- `i`： 模块 ID `moduleID` （ id 的简写）

	- `l`： 模块是否已注册（初始化为 `false`，loaded的简写）

	- `exprots`： 模块的输出（初始化为空）

- [第三部分](@TODO)，执行 `modlues` 中 `moduleId` 对应的模块立即执行函数，传入三个参数：

	- 刚才定义的对象 `module`

	- 对象的 `module.exports` 属性（传入的时候为空）

	- `__webpack_require__` 函数本身（方便在模块中调用其他模块）

- 执行完立即执行函数后，除了将模块本身的逻辑执行完，也会将模块的输出绑定到 `module.exports` 中

- [第四部分](@TODO)，将 `module.l` 设为 `true` 表示已加载

- [第五部分](@TODO)，最后输出 `module.exports` 属性

### `__webpack_require__` 中的绑定的属性和方法分析

- [m](@TODO): modules ，数组，所有的模块，即前文提到的 modules 参数

- [c](@TODO): cache ，对象，所有已安装的模块

- [d](@TODO): define ，函数，如果输出没有保存到模块的 `exports` 中，则使用 `Object.defineProperty` 将模块的输出保存到已安装模块的 `export` 属性中，会在模块中替换掉 `export` 语句。该函数包含三个参数：
	
	- exports: 模块的 `exports` 属性

	- name: 模块输出的代号（名字），默认为 `a` ，从 abcd往下排列

	- getter: 函数，返回模块的输出内容 

- [n](@TODO): 针对 `non-harmony` 模块的输出定义函数做一些兼容（这里我也不太理解）

- [o](@TODO): `Object.prototype.hasOwnProperty` 的 polyfill， 在 `__webpack_require__.d`中的判断是否这个输出是否已绑定到这个模块中用到

- [p](@TODO): 实际上就是配置文件中的 [`output.publicPath`](https://webpack.js.org/configuration/output/#output-publicpath)