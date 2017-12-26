---
layout: default
title: Markdown文档编辑，保存和展现
date: 2017-01-20 03:08:00
categories: blog
tags: [markdown, "prism js"]
description: Markdown文档编辑，保存和展现
---


markdown（简称md）是目前比较流行的文档格式。github和coding.net主流代码网站都支持这种格式的文档。

我们自己的文档如果想写成这种格式，存储和展示还需要自己探索，这里我总结下自己的方法。

### 一、编辑

这个就很多了，有的IDE像intellij idea自己就集成了md编辑器，还有浏览器插件，在线的开源编辑器更是不计其数，这里就不说了。

### 二、存储

 无论是存在数据库还是文本文件，我都建议直接存储md文件本身的内容，而不是存储转换后的html，原因就是不方便二次编辑。当然你也可以把两种格式都保存下来。
### 三、展现

#### 方法一、使用一些服务端的三方代码，把md转成html再输出到浏览器。

比较方便的三方代码，比如<https://github.com/SegmentFault/HyperDown>
#### 方法二、使用js

我选用的是这种方式：markdown-it + Prism

markdown-it是个md转html的js插件，地址<https://github.com/markdown-it/markdown-it>

官方文档很简单
```javascript
var md = window.markdownit();
var result = md.render('# markdown-it rulezz!');
```
但是这里有一个陷阱，如何把md的内容输入到这个text变量。

1. 如果直接输出php变量的话，需要对换行符和引号进行转义。

2. 使用ajax，请求plain text类型的内容，这个方法需求请求两次服务器。

3. 输出到html标签，再使用dom.innerHTML属性。但是innerHTML属性是经过HTML转义过的，需要decode回来。代码如下
```javascript
var md = window.markdownit();
var dom = document.getElementById('md-content');
dom.innerHTML = md.render(html_decode(dom.innerHTML));
function html_decode(str)
{
	var s;
	if (str.length == 0) return "";
	s = str.replace(/&lt;/g, "<");
	s = s.replace(/&gt;/g, ">");
	s = s.replace(/&nbsp;/g, " ");
	s = s.replace(/&#39;/g, "\'");
	s = s.replace(/&quot;/g, "\"");
	s = s.replace(/&amp;/g, "&");
	return s;
}
```
最后再加上prism.js [http://prismjs.com/](http://prismjs.com/ "prismjs")添加代码高亮效果就完成了