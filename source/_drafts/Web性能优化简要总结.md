---
title: Web性能优化简要总结
tags:
---
{% asset_img loading.jpg %}
基于HTTP协议的Web应用已经成为互联网的核心，优化Web应用性能、改善用户体验成为了非常重要的问题。
<!--more-->
# 性能指标
优化性能，首先要有衡量性能的指标，Chrome提供了十分科学高效的性能测量工具，Waterfall。
在Chrome的DevTool中，NetWork选项卡里就有所有请求的性能指标。（在打开DevTool后的请求才会被记录）
{% asset_img DevTool.png %}
上面的图是访问`baidu.com`的结果。
因为之前已经有过访问历史，所以需要做出两点改进，以模拟用户第一次访问时的场景。
首先，需要清除掉浏览器DNS缓存及本地DNS缓存。
DNS又叫做域名系统，也就是各个网站的网址。与ip地址相比，域名方便人们记忆和传播，可以说域名是ip地址的助记符。而机器并不能直接识别域名，所以需要通过DNS协议来查找域名对应的真实的ip地址。在每次尝试访问新的域名时，浏览器都会向网络服务提供商询问所要访问域名的ip地址，如果也没有找到，就会通过递归或迭代的方式向更上级的DNS服务器询问，直至找到结果。在找到结果后浏览器和操作系统会将这个结果缓存下来，在下次查询同样的域名时更快的返回结果。
然后，需要勾选Disable cache。浏览器同样会将访问过的静态资源缓(js/css/img...)存到本地，以便下一次打开该网站时更快的加载。
在做好准备工作时，我们可以选择最开始的一条请求，观察它的Waterfall数据。
{% asset_img WaterFall.png %}
在Google的开发者网站上，我们可以逐条找到这些条目的含义。
*Queueing：请求处于队列。当前有更高优先级的请求或者该域名已经有六个TCP连接时会出现。*
*Stalled：任何使请求处于队列的原因同样也会使请求阻塞。*
*DNS Lookup：浏览器正在解析域名到对应的ip地址。*
*Proxy negotiation: 浏览器正在与代理服务器协商。当网站后端使用了反向代理服务器时，会出现该情况。*
*Request sent: 浏览器正在发送请求。*
*Service Worker Preparation： 浏览器正在启动Service Worker。*
*Request to Service Worker： 浏览器正在向Service Worker发送请求。*
*Waiting（TTFB）：浏览器正在等待响应的第一个字节。*
*Content Download：浏览器正在接收响应。*
有一部分字段在开发者网站上也没有标注，不过根据名字我们就可以了解它们的含义。
例如SSL代表SSL的握手阶段耗时。
下面我们对于每一个字段仔细探讨一下优化策略。
# 加载优化
我们将`Queue, Stalled, Proxy negotiation, DNS Lookup, Initial connection, SSL, Request sent, Content Download, Service Worker`都视为加载阶段。
## Queue & Stalled
由开发者网站我们可以了解到，Queue & Stalled主要由两个原因，一个是此时有更高优先级的连接，另一个是浏览器对同一个域名最多会有六个TCP连接。
对于来自更高优先级连接的阻塞我们没有更好的办法，连接的优先级并不能手动来控制，例如VoIP（实时语音通话）的优先级势必会比打开一个网页要高很多。
浏览器对于一个同一个域名下的资源最多有六个并行的TCP连接，这会节省服务器的内存，也会减少线路上的拥塞程度。但作为网站，希望尽可能的利用带宽，那就需要用到`domain sharing`域名发散。所谓发散，就是将网站的资源分散到多个域名，以避免浏览器的单域名并行限制。这就是大多数网站都会有单独的`static`静态资源域名。
# 后端优化
# 前端优化