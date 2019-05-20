####  动态网站SEO解决方案汇总
  先撸撸几个概念：
- SPA：单页面应用，基于vue框架开发的项目很多都属于单页面应用。
- SSR ：server side rendering, 服务端渲染。
- SEO：搜索引擎优化，指通过对网站进行站内优化、修复和站外优化，从而提高网站的网站关键词排名以及公司产品的曝光度。
- Prerender：预渲染，Prerender.io是基于Node.js的程序，它可以让你的JavaScript网站支持搜索引擎，社交媒体，并且它兼容所有的JavaScript框架和库。它采用PhantomJS渲染JavaScript的网页然后呈现为HTML。此外，我们可以实现的prerender服务层来缓存访问过的页面，这将大大提高性能。（省事儿）
- Nuxt：是一个基于 Vue.js 的通用应用框架，预设了利用Vue.js开发服务端渲染的应用所需要的各种配置，可以为基于 Vue.js 的应用提供生成对应的静态站点的功能。
- Next:对标的是 React的通用应用框架，预设了React.js 开发服务端渲染的应用所需要的各种配置。
  
####	技术选型
- 结合现有项目框架选用，时间成本，学习成本进行合适的评估
- 从自身能力出发 如果涉及太多的服务端处理的东西 可以考虑运维层进行处理采用prerender
- 业务应用场景，业务线比较复杂工期较短的时候 建议自己进行PrerenderIo 的部署，使用自己的服务器进行对爬虫页面进行缓存。

####	三个技术选型优劣对比
- Next => React 文档大部分是英文的 配置项简单易上手，部署方便，大型的官网项目比较适合，用户交互复杂的时候采用Next 进行项目开发。
- Nuxt => Vue 基本是Next的翻版，语法也是Next语法，大坑的地方是在 大多数稳定的项目是1.4.2的版本 现有2.X的版本 基本完全不兼容老版本。
  - 渲染效率比较低，业务复杂的时候 编译速度十分慢。十分慢
  - 版本跨度大的适合兼容性低。
- PhantomJS 原理就是通过Nginx配置，将搜索引擎的爬虫请求转发到一个node server，再通过PhantomJS来解析完整的HTML。
  - 可以做为一整套通用服务，所有的SPA页面基本不需要二次重构。
  - 缺点是受网络波动制约性比较强，
  - 适合复杂项目短时间进行收录处理
  - 需要网络层的权限 需要和运维进行沟通。

<b>整体还是结合当前需求的场景 和自身的条件来进行选择,短时间高效完成需求。</b>

相关收录文章：
>[Nuxt](https://zh.nuxtjs.org/guide/)
[Next](https://nextjs.org/)
[Prerender + vuejs SEO最佳实践 百度爬虫 + 谷歌收录 亲测成功](https://www.deboy.cn/prerender-vuejs1-X-SEO-best-practice.html)[Prerender.io](https://blog.csdn.net/niuniuasb/article/details/60957810)
[前端渲染与 SEO 优化踩坑 小记](https://www.v2ex.com/t/302616)
[用PhantomJS来给AJAX站点做SEO优化](http://f2er.info/article/29)
####一个PhantomJS任务脚本
首先，我们需要一个叫spider.js的文件，用于phantomjs 解析网站。
```js
"use strict";

// 单个资源等待时间，避免资源加载后还需要加载其他资源
var resourceWait = 500;
var resourceWaitTimer;
// 最大等待时间
var maxWait = 5000;
var maxWaitTimer;
// 资源计数
var resourceCount = 0;
// PhantomJS WebPage模块
var page = require('webpage').create();
// NodeJS 系统模块
var system = require('system');
// 从CLI中获取第二个参数为目标URL
var url = system.args[1];
// 设置PhantomJS视窗大小
page.viewportSize = {
	width: 1280,
	height: 1014
};
// 获取镜像
var capture = function(errCode){
	// 外部通过stdout获取页面内容
	console.log(page.content);
	// 清除计时器
	clearTimeout(maxWaitTimer);
	// 任务完成，正常退出
	phantom.exit(errCode);
};
// 资源请求并计数
page.onResourceRequested = function(req){
	resourceCount++;
	clearTimeout(resourceWaitTimer);
};
// 资源加载完毕
page.onResourceReceived = function (res) {
	// chunk模式的HTTP回包，会多次触发resourceReceived事件，需要判断资源是否已经end
	if (res.stage !== 'end'){
	    return;
	}
	resourceCount--;
	if (resourceCount === 0){
		// 当页面中全部资源都加载完毕后，截取当前渲染出来的html
		// 由于onResourceReceived在资源加载完毕就立即被调用了，我们需要给一些时间让JS跑解析任务
		// 这里默认预留500毫秒
		resourceWaitTimer = setTimeout(capture, resourceWait);
	}
};
// 资源加载超时
page.onResourceTimeout = function(req){
	resouceCount--;
};
// 资源加载失败
page.onResourceError = function(err){
	resourceCount--;
};
// 打开页面
page.open(url, function (status) {
	if (status !== 'success') {
		phantom.exit(1);
	} else {
		// 当改页面的初始html返回成功后，开启定时器
		// 当到达最大时间（默认5秒）的时候，截取那一时刻渲染出来的html
		maxWaitTimer = setTimeout(function(){
			capture(2);
		}, maxWait);
	}
});
```
进行测试=> `phantomjs spider.js 'https://www.baidu.com/'`
#### 命令服务化
响应搜索引擎爬虫的请求，我们需要将此命令服务化，通过node起个简单的web服务
```js
var express = require('express');
var app = express();
// 引入NodeJS的子进程模块
var child_process = require('child_process');
app.get('/', function(req, res){
    // 完整URL
    var url = req.protocol + '://'+ req.hostname + req.originalUrl;
    console.log(req,req.hostname)
    // 预渲染后的页面字符串容器
    var content = '';
    // 开启一个phantomjs子进程
    var phantom = child_process.spawn('phantomjs', ['spider.js', url]);
    
    // 设置stdout字符编码
    phantom.stdout.setEncoding('utf8');
    // 监听phantomjs的stdout，并拼接起来
    phantom.stdout.on('data', function(data){
        content += data.toString();
    });
    // 监听子进程退出事件
    phantom.on('exit', function(code){
        switch (code){
            case 1:
                console.log('加载失败');
                res.send('加载失败');
                break;
            case 2:
                console.log('加载超时: '+ url);
                res.send(content);
                break;
            default:
                res.send(content);
                break;
        }
    });
    
});
app.listen(3002)
```
运行node server.js，此时我们已经有了一个预渲染的web服务啦，接下来的工作便是将搜索引擎爬虫的请求转发到这个web服务，最终将渲染结果返回给爬虫。
为了防止node进程挂掉，可以使用nohup来启动，nohup node server.js &。
通过Nginx配置，我们可以轻松的解决这个问题。
```ssh
# 定义一个Nginx的upstream为spider_server
upstream spider_server {
  server localhost:3000;
}
# 指定一个范围，默认 / 表示全部请求
location / {
  proxy_set_header  Host            $host:$proxy_port;
  proxy_set_header  X-Real-IP       $remote_addr;
  proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
  # 当UA里面含有Baiduspider的时候，同时可以加其他的头信息进行转发 流量Nginx以反向代理的形式，将流量传递给spider_server
  if ($http_user_agent ~* "Baiduspider") {
    proxy_pass  http://spider_server;
  }
}
```
>参考链接：
https://www.mxgw.info/t/phantomjs-prerender-for-seo.html
http://imweb.io/topic/560b402ac2317a8c3e08621c
https://icewing.cc/linux-install-phantomjs.html
https://www.jianshu.com/p/2bbbc2fcd16d
