---
title: webpack & webpack-dev-server简介
date: 2017-04-22 09:24:44
tags: 
- 前端
- Javascipt
---
{% asset_img what-is-webpack.png %}

# 如其官网所说，Webpack用于打包含有依赖关系的模块。
在了解一个工具前，我们应该知道为什么使用它。
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
<script src="script2.js"></script>
<script src="script3.js"></script>
<script src="script4.js"></script>
<script src="script5.js"></script>
<script src="script6.js"></script>
<script src="script7.js"></script>
<script src="script8.js"></script>
{% endcodeblock %}
这样会带来一些问题
首先，尽管浏览器会确保各个脚本按顺序执行，但却无法判断其依赖关系。
{% asset_img dependence.png %}
当各个脚本间有着非常复杂的依赖关系时，组织它们的加载顺序就变得很有难度。
假如script8.js仅依赖script2.js就可以执行，但script8.js同样需要等待其他所有脚本都加载执行完成才可以执行。
那为什么不能直接把script8.js提到script3.js前面呢？这会带来新的问题，如果script8.js并不是一个优先级很高的脚本，或者说在页面加载完成后很长一段时间并不会用到它，那么优先加载它就会影响其他更重要的脚本的加载，从而损害体验。
另外，如果脚本间存在着依赖，那么被依赖的脚本必须通过声明全局变量来导出被依赖的变量，以供其他脚本使用，这样做会污染全局变量。
例如，jQuery通过给window对象赋值jQuery属性来导出供其他脚本使用的对象，但其他的脚本不能够再声明名为jQuery的变量。
所以，我们需要一种“声明依赖，动态加载”的方式来加载所有需要的脚本。AMD/CMD规范可以做到这样的效果。
{% codeblock script1.js lang:javascript %}
var global1 = "king";
console.log(global3);
{% endcodeblock %}
{% codeblock script2.js lang:javascript %}
var global2 = global1 + "of the";
{% endcodeblock %}
{% codeblock script3.js lang:javascript %}
var global3 = global2 + "world";
{% endcodeblock %}
在上述代码中