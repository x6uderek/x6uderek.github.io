---
layout: page
title: 利用gb2312编码生成随机汉字
date: 2017-05-10 08:18:00
categories: blog
tags: [bg2312]
description: 利用gb2312编码生成随机汉字
---

有时候会遇到随机汉字的需求，比如汉字验证码，随机昵称等，可以利用gb2312编码规则获取常用的随机汉字
# GB2312编码规则
GB2312编码对所收录字符进行了“分区”处理，共94个区，每区含有94个位，共8836个码位。这种表示方式也称为区位码。

* 01-09区收录除汉字外的682个字符。
* 10-15区为空白区，没有使用。
* 16-55区收录3755个一级汉字，按拼音排序。
* 56-87区收录3008个二级汉字，按部首/笔画排序。
* 88-94区为空白区，没有使用。

举例来说，“啊”字是GB2312编码中的第一个汉字，它位于16区的01位，所以它的区位码就是1601。

GB2312编码范围：A1A1－FEFE，其中汉字的编码范围为B0A1-F7FE，第一字节0xB0-0xF7（对应区号：16－87），第二个字节0xA1-0xFE（对应位号：01－94）。

以下程序输出gb2312编码中所有的汉字：
```php
for($i = 0xB0; $i <= 0xF7; $i++) {
	echo sprintf("%X:",$i);
	for($j = 0xA1; $j <= 0xFE; $j++) {
		if($i==0xD7 && $j>=0xFA) {  //D7 后面缺5个，没有对应汉字。
			break;
		}
		$a = hex2bin(sprintf("%X%X",$i,$j));
		echo iconv('gb2312','utf-8',$a);
	}
	echo '<br/>';
}
```

以下方法可以产生一个随机常用字：
```php
$qu = rand(0xB0, 0xD7); //D7以后都是不常用汉字
if($qu == 0xD7) {
	$wei = rand(0xA1, 0xF9);
}
else {
	$wei = rand(0xA1, 0xFE);
}
echo iconv('gb2312','utf-8',hex2bin(sprintf("%X%X",$qu,$wei)));
```

参考资料：
1. [GB2312 编码][1]

[1]: <http://www.qqxiuzi.cn/zh/hanzi-gb2312-bianma.php>