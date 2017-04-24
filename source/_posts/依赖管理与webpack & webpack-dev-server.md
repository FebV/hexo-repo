---
title: 依赖管理与webpack & webpack-dev-server
date: 2017-04-22 09:24:44
tags: 
- 前端
- JavaScipt
---
{% asset_img what-is-webpack.png %}
# 依赖管理
如其官网所说，Webpack用于打包含有依赖关系的模块。
在了解一个工具前，我们应该知道为什么使用它。
## Tradition
在一开始，前端项目的规模并不大，所以靠script标签来引入JavaScript代码。
{% codeblock index.html lang:html %}
<script src="script.js"></script>
{% endcodeblock %}
在代码需要依赖外部类库时，就需要让类库的代码先被加载执行。
{% codeblock index.html lang:html %}
<script src="https://static.xxx.com/js/jQuery.min.js"></script>
<script src="script.js"></script>
{% endcodeblock %}
当自己的代码发展到几万行时，使用一个文件就会变得捉襟见肘，既不方便协同开发，也不利于使用缓存策略，所以自然要分拆成多个脚本文件。
{% codeblock index.html lang:html %}
<script src="https://static.xxx.com/js/jQuery.min.js"></script>
<script src="script1.js"></script>
<script src="script4.js"></script>
<script src="script6.js"></script>
<script src="script2.js"></script>
<script src="script7.js"></script>
<script src="script5.js"></script>
<script src="script8.js"></script>
<script src="script3.js"></script>
{% endcodeblock %}
这样会带来一些问题
首先，尽管浏览器会确保各个脚本按顺序执行，但却无法判断其依赖关系。
{% asset_img dependence.png %}
当各个脚本间有着非常复杂的依赖关系时，组织它们的加载顺序就变得很有难度。
另外，假如script5.js仅依赖script1.js就可以执行，但script5.js同样需要等待所有在它之前引用的脚本都加载执行完成才可以执行。
那为什么不能直接把script5.js提到script1.js的后面呢？这会带来新的问题，如果script5.js并不是一个优先级很高的脚本，或者说在页面加载完成后很长一段时间并不会用到它，那么优先加载它就会影响其他更重要的脚本的加载，从而损害体验。
而且，如果脚本间存在着依赖，那么被依赖的脚本必须通过声明全局变量来导出被依赖的变量，以供其他脚本使用，这样做会污染全局变量。
例如，jQuery通过给window对象赋值jQuery属性来导出供其他脚本使用的对象，但其他的脚本不能够再声明名为jQuery的变量。
## AMD/CMD
我们需要一种“声明依赖，动态加载”的方式来加载所有需要的脚本。AMD/CMD规范可以做到这样的效果，简单介绍AMD规范最流行的实现：require.js
{% codeblock script1.js lang:js %}
define(['jQuery'], function(jQuery){
　　　　function do(){
　　　　　　jQuery.doSomething();
　　　　}
　　　　return {
            do: do
　　　　};
　　});
{% endcodeblock %}
{% codeblock script5.js lang:js %}
require(['script1'], function (script1){
　　　　// code rely on script1
});
{% endcodeblock %}
在上述代码中，script1.js依赖于jQuery，并声明成为了一个模块，而script5.js的执行又依赖于script1.js，因此require.js会首先加载jQuery，然后加载script1.js，最后加载script5.js，做到了“声明依赖，动态加载”的模块化。
## CommonJS
同样也介绍一下服务端JavaScript的依赖管理规范CommonJs
在服务端，加载依赖都是通过文件I/O，不像在浏览器端需要通过网络请求，所以在加载速度和稳定性上远远超过浏览器环境，也就没有必要通过异步方式加载所依赖的模块。其写法如下：
{% codeblock index.js lang:js %}
var user = require('./user');
console.log(user.username);
{% endcodeblock %}
{% codeblock user.js lang:js %}
var user = {
    username: "FebV",
    password: "mypass",
};
module.exports = user;
{% endcodeblock %}
## ES6
为了统一浏览器与服务端的模块化风格，ES6推出了import/export语法：
{% codeblock index.js lang:js %}
import user from './user';
console.log(user.username);
{% endcodeblock %}
{% codeblock user.js lang:js %}
var user = {
    username: "FebV",
    password: "mypass",
};
export default user;
{% endcodeblock %}
统一了标准，前后端模块就可以共用同一套规范来引入依赖，有利于模块的复用。
很遗憾，无论是在浏览器端还是在服务端，都不能够直接使用ES6的模块化语法。
于是，webpack便适时登场了。
# Webpack
由于ES6风格与CommonJs十分相似，所以通过CommonJs的语法也能够比较顺利地引入使用ES6规范声明的模块，webpack主要用于支持浏览器端的模块化语法。
## 安装webpack
通过npm，我们可以轻松地为项目添加webpack依赖。
{% codeblock project/root/ %}
npm install webpack --save-dev
{% endcodeblock %}
## 使用webpack
推荐使用独立的webpack.config.js文件作为webpack的配置文件
{% codeblock project/root/webpack.config.js %}
module.exports = {
    entry: "./index.js"
    output: {
        filename: "bundle.js",
        path: "__dirname",
    }
}
{% endcodeblock %}
上面的配置文件展示了webpack运行所必须的最简配置项，由于webpack使用Node作为运行环境，很明显其配置文件使用了CommonJs风格。
entry   表明项目的入口文件，即Js代码从何处开始执行。
output  描述了打包后的目标文件，包括其文件名及输出路径。
## webpack原理
webpack将从入口文件开始，把所有的依赖打包成一个bundle.js，其运行效果等同于在ES6规范下期望的运行效果，以前文中ES6风格模块的代码为例，使用webpack将其打包之后的bundle.js如下：
{% codeblock bundle.js lang:js %}
/******/ (function(modules) { // webpackBootstrap
/******/ 	// The module cache
/******/ 	var installedModules = {};
/******/
/******/ 	// The require function
/******/ 	function __webpack_require__(moduleId) {
/******/
/******/ 		// Check if module is in cache
/******/ 		if(installedModules[moduleId]) {
/******/ 			return installedModules[moduleId].exports;
/******/ 		}
/******/ 		// Create a new module (and put it into the cache)
/******/ 		var module = installedModules[moduleId] = {
/******/ 			i: moduleId,
/******/ 			l: false,
/******/ 			exports: {}
/******/ 		};
/******/
/******/ 		// Execute the module function
/******/ 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
/******/
/******/ 		// Flag the module as loaded
/******/ 		module.l = true;
/******/
/******/ 		// Return the exports of the module
/******/ 		return module.exports;
/******/ 	}
/******/
/******/
/******/ 	// expose the modules object (__webpack_modules__)
/******/ 	__webpack_require__.m = modules;
/******/
/******/ 	// expose the module cache
/******/ 	__webpack_require__.c = installedModules;
/******/
/******/ 	// identity function for calling harmony imports with the correct context
/******/ 	__webpack_require__.i = function(value) { return value; };
/******/
/******/ 	// define getter function for harmony exports
/******/ 	__webpack_require__.d = function(exports, name, getter) {
/******/ 		if(!__webpack_require__.o(exports, name)) {
/******/ 			Object.defineProperty(exports, name, {
/******/ 				configurable: false,
/******/ 				enumerable: true,
/******/ 				get: getter
/******/ 			});
/******/ 		}
/******/ 	};
/******/
/******/ 	// getDefaultExport function for compatibility with non-harmony modules
/******/ 	__webpack_require__.n = function(module) {
/******/ 		var getter = module && module.__esModule ?
/******/ 			function getDefault() { return module['default']; } :
/******/ 			function getModuleExports() { return module; };
/******/ 		__webpack_require__.d(getter, 'a', getter);
/******/ 		return getter;
/******/ 	};
/******/
/******/ 	// Object.prototype.hasOwnProperty.call
/******/ 	__webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
/******/
/******/ 	// __webpack_public_path__
/******/ 	__webpack_require__.p = "";
/******/
/******/ 	// Load entry module and return exports
/******/ 	return __webpack_require__(__webpack_require__.s = 1);
/******/ })
/************************************************************************/
/******/ ([
/* 0 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
var user = {
    username: "FebV",
    password: "mypass"
}

/* harmony default export */ __webpack_exports__["a"] = (user);

/***/ }),
/* 1 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__user__ = __webpack_require__(0);

console.log(__WEBPACK_IMPORTED_MODULE_0__user__["a" /* default */].username);

/***/ })
/******/ ]);
{% endcodeblock %}
可以看到，webpack打包后的bundle.js通过一个匿名的立即执行函数来加载模块，下面我们逐行分析这段代码。
{% codeblock line 1-3 lang:js %}
 (function(modules) { // webpackBootstrap
 	// The module cache
 	var installedModules = {};
{% endcodeblock %}
在括号中声明的匿名函数很明显是一个需要立即执行的函数，使用这种方式可以不声明全局变量，也就不污染命名空间。jQuery就是以这样的方式执行，只在执行过程中通过`window.jQuery = jQuery`来对外暴露对象。
`installedModules`变量则代表了已经加载的模块。加载模块需要执行模块中的代码，如果能将模块要导出的变量缓存起来，那么第二次加载此模块就可以缓存的值而无需再一次执行模块中的代码。
{% codeblock line 5-27 lang:js %}
// The require function
/******/ 	function __webpack_require__(moduleId) {
/******/
/******/ 		// Check if module is in cache
/******/ 		if(installedModules[moduleId]) {
/******/ 			return installedModules[moduleId].exports;
/******/ 		}
/******/ 		// Create a new module (and put it into the cache)
/******/ 		var module = installedModules[moduleId] = {
/******/ 			i: moduleId,
/******/ 			l: false,
/******/ 			exports: {}
/******/ 		};
/******/
/******/ 		// Execute the module function
/******/ 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
/******/
/******/ 		// Flag the module as loaded
/******/ 		module.l = true;
/******/
/******/ 		// Return the exports of the module
/******/ 		return module.exports;
/******/ 	}
{% endcodeblock %}
这段代码在立即执行函数的内部定义了`__webpack_require__`函数，通过名称我们可以了解到这个函数主要用来加载模块。可以看到，如果`installedModules`数组中已经存在了要加载的模块，那么就可以直接返回对应模块的输出变量；如果不存在，那就向`installedModules`中添加这个模块，并设置其标识`i`为其id，是否已被加载`l`为false，输出变量为空对象。之后通过`call`函数执行这个模块中的代码，即加载这个模块。在后面可以看到，模块中的代码被webpack封装在一个匿名函数中。在加载完成后，模块的是否被加载属性`l`被设置为true，最后这个函数将返回模块的输出变量。
{% codeblock line 30- lang:js %}
/******/ 	// expose the modules object (__webpack_modules__)
/******/ 	__webpack_require__.m = modules;
/******/
/******/ 	// expose the module cache
/******/ 	__webpack_require__.c = installedModules;
/******/
/******/ 	// identity function for calling harmony imports with the correct context
/******/ 	__webpack_require__.i = function(value) { return value; };
/******/
/******/ 	// define getter function for harmony exports
/******/ 	__webpack_require__.d = function(exports, name, getter) {
/******/ 		if(!__webpack_require__.o(exports, name)) {
/******/ 			Object.defineProperty(exports, name, {
/******/ 				configurable: false,
/******/ 				enumerable: true,
/******/ 				get: getter
/******/ 			});
/******/ 		}
/******/ 	};
/******/
/******/ 	// getDefaultExport function for compatibility with non-harmony modules
/******/ 	__webpack_require__.n = function(module) {
/******/ 		var getter = module && module.__esModule ?
/******/ 			function getDefault() { return module['default']; } :
/******/ 			function getModuleExports() { return module; };
/******/ 		__webpack_require__.d(getter, 'a', getter);
/******/ 		return getter;
/******/ 	};
/******/
/******/ 	// Object.prototype.hasOwnProperty.call
/******/ 	__webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
/******/
/******/ 	// __webpack_public_path__
/******/ 	__webpack_require__.p = "";
{% endcodeblock %}
得益于JavaScript一切皆对象的概念和动态属性的特性，即使函数本身也可以被随意添加属性。在上面的代码中我们可以看到，webpack给`__webpack_require__`函数添加了一些属性，例如所有的模块`m`、已经被加载的模块`c`和一些与ES6模块相关的函数等等。webpack内部默认以CommonJs规范实现，所以遇到ES6模块需要做一些微调，这里便不再赘述。