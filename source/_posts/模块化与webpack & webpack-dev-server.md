---
title: 模块化与webpack & webpack-dev-server
date: 2017-04-22 09:24:44
tags: 
- 前端
- JavaScript
---
{% asset_img what-is-webpack.png %}
# 模块化
如其官网所说，Webpack用于打包含有依赖关系的模块。
在了解一个工具前，我们应该知道为什么使用它。
<!--more-->
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
# Webpack介绍
由于ES6风格与CommonJs十分相似，所以通过CommonJs的语法也能够比较顺利地引入使用ES6规范声明的模块，webpack主要用于支持浏览器端的模块化语法。
## 安装webpack
通过npm，我们可以轻松地为项目添加webpack依赖。
{% codeblock project/root/ %}
npm install webpack --save-dev
{% endcodeblock %}
## 初步使用webpack
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
## bundle.js分析
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
{% codeblock line 30-63 lang:js %}
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
得益于JavaScript一切皆对象的概念和动态属性的特性，即使函数本身也可以被随意添加属性。在上面的代码中我们可以看到，webpack给`__webpack_require__`函数添加了一些属性，例如所有的模块`m`、已经被加载的模块`c`和一些与ES6模块相关的函数等等。webpack内部默认以CommonJs规范实现，所以遇到ES6模块需要做一些微调，这里便不再赘述。在加载模块时`__webpack_require__`函数会作为参数被传递到被加载的模块，从而将这些属性和函数暴露给模块内部。
{% codeblock line 65-66 lang:js %}
/******/ 	// Load entry module and return exports
/******/ 	return __webpack_require__(__webpack_require__.s = 1);
{% endcodeblock %}
在匿名函数的最后，调用了`__webpack_require__`函数，将此函数也作为一个模块加载到modules中。
{% codeblock line 69-81 lang:js %}
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
{% endcodeblock %}
在此处，我们编写的user.js被封装到一个匿名函数中，并作为入口匿名立即执行函数的参数传入。ES6模块语法规定在模块的内部自动使用严格模式。在模块代码执行完成之后`__webpack_exports__["a"] = (user);`将模块的输出变量赋值给`__webpack_exports__`以供其他模块使用。
{% codeblock line 83-92 lang:js %}
/******/ ([
/* 0 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {
/***/ (function(module, __webpack_exports__, __webpack_require__) {
"use strict";
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__user__ = __webpack_require__(0);
console.log(__WEBPACK_IMPORTED_MODULE_0__user__["a" /* default */].username);
/***/ })
/******/ ]);
{% endcodeblock %}
在webpack打包之后index.js模块也被封装到一个匿名函数中。在index.js中我们使用import引入了user.js模块，在打包后的模块代码中，首先声明其为esModule，方便在其他模块中识别此模块的模块规范。因为index.js模块依赖于user.js，所以调用`__webpack_require__`来加载user.js，然后就可以使用user.js中输出的变量了。
## 配置webpack
虽然webpack一开始被用来打包互相依赖的模块，但令webpack大放光芒的是它提供的丰富的功能，使用webpack可以极大提高前端开发的效率。
### loader
通过loader，webpack允许在打包前首先对JavaScript代码等静态资源进行处理，以babel为例。
JavaScript的ES6标准为JavaScript添加了一系列十分有用的函数和语法，能够让编写前端代码更加流畅优雅，但现阶段浏览器并不能够完全支持ES6标准的语法，如果既希望使用ES6的语法又希望能够在大部分浏览器上不会又兼容性问题，我们就需要使用babel将ES6代码转为ES5的标准。
`npm install --save-dev babel-core babel-loader babel-preset-es2015 `
{% codeblock project/root/webpack.config.js %}
module.exports = {
    entry: "./index.js",
    output: {
        path: __dirname,
        filename: 'bundle.js',
    },
    module: {
        loaders: [
            {
                test: /\.js$/,
                exclude: /node_modules/,
                loader: "babel-loader"
            }
        ],
    }
}
{% endcodeblock %}
{% codeblock project/root/.babelrc %}
{
    "presets": ["es2015"]
}
{% endcodeblock %}
我们声明一个ES6标准中的箭头函数
{% codeblock index.js lang:js %}
var func = () => {
    console.log('func');
}
{% endcodeblock %}
我们找到这个模块转码后所在的位置
{% codeblock bundle.js line 76-78 lang:js %}
var func = function func() {
    console.log('func');
};
{% endcodeblock %}
其变成了ES5标准，可以在几乎所有的流行浏览器中使用。
### plugin
webpack同样还带有很多非常有用的plugin，我们以uglifyJsPlugin为例
{% codeblock project/root/webpack.config.js %}
var webpack = require('webpack');
module.exports = {
    entry: "./index.js",
    output: {
        path: __dirname,
        filename: 'bundle.js',
    },
    module: {
        loaders: [
            {
                test: /\.js$/,
                exclude: /node_modules/,
                loader: "babel-loader"
            }
        ],
    },
    plugins: [
        new webpack.optimize.UglifyJsPlugin()
    ],
}
{% endcodeblock %}
{% codeblock bundle.js %}
!function(t){function n(e){if(r[e])return r[e].exports;var o=r[e]={i:e,l:!1,exports:{}};return t[e].call(o.exports,o,o.exports,n),o.l=!0,o.exports}var r={};n.m=t,n.c=r,n.i=function(t){return t},n.d=function(t,r,e){n.o(t,r)||Object.defineProperty(t,r,{configurable:!1,enumerable:!0,get:e})},n.n=function(t){var r=t&&t.__esModule?function(){return t.default}:function(){return t};return n.d(r,"a",r),r},n.o=function(t,n){return Object.prototype.hasOwnProperty.call(t,n)},n.p="",n(n.s=0)}([function(t,n,r){"use strict"}]);
{% endcodeblock %}
UglifyJsPlugin可以将bundle.js中的空格和换行都删去，并将函数名和变量名换为很短的字符串，从而缩减bundle.js文件的体积，较小的体积可以改善页面加载时间。

# Webpack-dev-server
{% asset_img file_pro.png %}
在编写好html文档之后，双击html文件即可在浏览器中打开，这样十分方便，但会有一些问题。
直接双击打开是使用的file协议，而非一般的http协议，在file协议下，例如aJax等操作都会因为浏览器的安全策略受限，所以我们需要搭建一个本地服务器，以使用JavaScript的全部功能，webpack-dev-server就可以作为本地服务器。
## HotModuleReplacement
webpack-dev-server除了作为静态文件服务器之外，还有很多功能，例如自动刷新的功能。平时我们在修改完JavaScript代码后，需要手动刷新浏览器才能看到新代码的执行效果。webpack可以在我们保存JavaScript文件时自动刷新浏览器。
{% asset_img websocket-hot.png  %}
实现的原理是在页面中嵌入js脚本，使用websocket与webpack-dev-server相连，当webpack-dev-server检测到代码文件变动时，就重新打包，然后通过websocket通知浏览器刷新页面。
{% codeblock project/root/webpack.config.js %}
var webpack = require('webpack')

module.exports = {
    entry: "./index.js",
    output: {
        filename: "bundle.js",
        path: __dirname
    },
    devServer: {
        hot: true
    },
    module: {
        loaders: [
            {
                test: /\.js$/,
                exclude: /node_modules/,
                loader: "babel-loader"
            }
        ]
    },
    plugins: [
        new webpack.HotModuleReplacementPlugin(),
        new webpack.optimize.UglifyJsPlugin()
    ],
}
{% endcodeblock %}

## HistoryApiFallback & Proxy
devServer还有两个常用的属性：historyApiFallback & proxy

historyApiFallback主要用于单页面应用刷新时404问题。
在单页面应用中，url由前端管理，当处于非首页url时，直接刷新会请求对应url的页面，这样的url在后端是不存在的，所以会遇到404的错误。当historyApiFallback置为true时，访问404页面webpack-dev-server同样会响应index.html，此时前端路由会处理url呈现相应的视图。

proxy用于反向代理。
{% codeblock %}
proxy: {
    '/api': {
        target: 'http://localhost:3000/',
    }
}
{% endcodeblock %}
通常提供API的后端应用与静态资源服务器时分离的，这样请求后端API是仍然会遇到跨域的问题。可以采取配置proxy将所有到/api路径下的请求转发到其他的地址，这样就避免了前端跨域的问题。
webpack & webpack-dev-server还有很多别的功能，这里先不赘述，对此感兴趣的话可以访问[webpack官方网站](https://webpack.js.org/)
# 综述
模块化是前端工程化的重要概念，如果不使用模块化那么构建可维护易扩展的大型前端项目无异于天方夜谭。webpack是现阶段解决模块化的优秀方案，也是前端工程化的重要工具，对于前端开发者来说十分有必要了解和使用。