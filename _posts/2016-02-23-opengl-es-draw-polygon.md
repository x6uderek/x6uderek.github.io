---
layout: page
title: OpenGL ES绘制多边形
date: 2016-02-23 17:51:00
categories: blog
tags: [Android,"OpenGl ES"]
description: OpenGL ES绘制多边形
---


OpenGL ES是精简版的OpenGL，没有绘制四边形的函数，因此只能由三角型拼出来。有两个绘制函数
1. 函数`glDrawArrays`，用法很简单，把需要绘制的三角形各个顶点放到一个大数组里面就行了。但是这样做会造成不必要的内存消耗，因为很多顶点坐标是重合的，却不能重复利用，因为它没有indices索引参数，因此要求多边形的顶点在vertexBuffer上是连续的，这个要求可能会与实际项目的数据结构冲突，因此不太好用。
2. 函数`glDrawElements`支持indices索引参数，并且支持3种不同类型的索引排列方式：`GL_TRIANGLES`、`GL_TRIANGLE_STRIP`和`GL_TRIANGLE_FAN`

下面着重讲述着3种index排序方式

### 1、`GL_TRIANGLES`

这种最简单，它表示indices索引数组里面每3个为一个三角形3顶点坐标在vertex数组中的位置。

例如：要绘制以下多边形

![多边形1](http://derekblog-upload.stor.sinaapp.com/2016_02/6474f75b6a16b08b5ee3fc0270ffd01f.png)

vertex数组中有v1-v5的坐标点，其index分别为0-4

那么使用GL_TRIANGLES指定indices参数时，应当为：
```java
[0,1,2,0,2,3,0,3,4]
```
### 2、`GL_TRIANGLE_FAN`

意思就是扇形排列的三角形

同样是上面的多边形例子，使用`GL_TRIANGLE_FAN`只需要指定indices为：
```java
[0,1,2,3,4]
```

因为它会把第一个顶点的作为公用顶点，这样就形成了[0,1,2][0,2,3][0,3,4]3个三角形

但是这种方式有些限制：如果需要绘制的三角形不能公用一个顶点，那么他就不能一次完成整个绘制。比如

![多边形2](http://derekblog-upload.stor.sinaapp.com/2016_02/274884a5cf4272fb97d554946387ed29.png)

这个多边形使用`GL_TRIANGLE_FAN`就不能一次绘制

### 3、`GL_TRIANGLE_STRIP`

意思就是条带状的三角形，这个比较复杂。

上面使用`GL_TRIANGLE_FAN`不能一次绘制的图形使用`GL_TRIANGLE_STRIP`只需要一次指定`v2,v3,v1,v4,v5,v6`的索引就行了，即`[1,2,0,3,4,5]`。

它会在indices索引里面每个索引都当作起始点，再向后取两个索引组成三角形，也就是说使用上一个三角形后面连个顶点作为顶点，每次只取一个新顶点。

上面的例子就形成了`[1,2,0][2,0,3][0,3,4][3,4,5]`4个三角形。

还有一个规则就是：序号为偶数的三角形前两个顶点位置互换。

那么上面的4个三角形实际上是：`[1,2,0][0,2,3][0,3,4][4,3,5]`，这样做是为了保证所有三角形与第一个三角形的顶点顺序一致（同为顺/逆时针）。

还有一个问题：如果需要绘制的图形不是条带状怎么办？如下图

![多边形3](http://derekblog-upload.stor.sinaapp.com/2016_02/c42f819c4ec698a510c5767fa9b79f92.png)

如果指定索引为`[0,4,1,5,2,6,3,7,4,8,5,9,6,10,7,11]`,那么将会产生预期之外的三角形：`[3,7,4][7,4,8]`

解决方法是插入额外的索引形成无法构建的三角形，OpenGL将自动跳过这些三角形，像这样：
```java
[0,4,1,5,2,6,3,7,7,4,4,8,5,9,6,10,7,11]
```
这会形成4个额外的三角形：`[3,7,7][7,7,4][7,4,4][4,4,8]`。OpenGL检测到这4个三角形有两个顶点时重合的，于是跳过他们。这样，通过在indices插入上一个条状图形的末尾顶点索引和下一个条状图形的首顶点，我们就能一次绘制多个条状图形了。

另外，如果指定上色类型为纯色`gl.glShadeModel(GL10.GL_FLAT)`，那么三角形的颜色将由最后一个顶点的颜色决定。

参考资料：

<http://www.learnopengles.com/android-lesson-eight-an-introduction-to-index-buffer-objects-ibos/>

<http://blog.csdn.net/xiajun07061225/article/details/7455283>