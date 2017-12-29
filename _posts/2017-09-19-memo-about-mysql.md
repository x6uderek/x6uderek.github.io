---
layout: page
title: 备忘录——mysql相关
date: 2017-09-19 08:55:00
categories: blog
tags: [mysql]
description: 备忘录——mysql相关
---

# POD 绑定数组参数

PDO无法直接绑定数组类型的参数，但是我们经常遇到，如下这种情况

```php
$ids = array(1,2,3);
$db = new PDO(...);
$stmt = $db->prepare('SELECT * FROM user WHERE id IN(:an_array)');
$stmt->bindParam('an_array',$ids);
$stmt->execute();
```
很不幸，我们想要的这种写法PDO是不支持的，因为PDO不知道怎样把数组里面的值，组织起来

这里是一种可用方案
```php
$ids     = array(1, 2, 3);
$inQuery = implode(',', array_fill(0, count($ids), '?'));
$db = new PDO(...);
$stmt = $db->prepare('SELECT * FROM user WHERE id IN(' . $inQuery . ')');
$stmt->execute($ids);
```

# POD 绑定LIKE参数

需要记住的简单的规则：like 后面的占位符，不能加 % _  或者引号，正确的做法是
```php
$sth = $dbh->prepare('SELECT * FROM books WHERE title LIKE :keyword');
$keyword = '%'.$keyword.'%';
$sth->bindParam(':keyword', $keyword, PDO::PARAM_STR);
```
