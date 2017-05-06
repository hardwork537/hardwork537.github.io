---
layout: post
title: php函数
category: php
tags: [php]
description: 一些魔术函数、常用函数
---

### __invoke
PHP自从5.3版以来就新增了一个叫做__invoke的魔术方法，使用该方法就可以在创建实例后，直接调用对象
例如：
```
$n = new testClass;
$n();
```

### __set($key, $value)
向一个难以访问的属性赋值的时候 __set() 方法被调用 
难以访问包括：（1）私有属性，（2）没有初始化的属性

### __get($key)
从一个难以访问的属性读取数据的时候 __get() 方法被调用
难以访问包括：（1）私有属性，（2）没有初始化的属性

### __clone
在clone $class时会被调用
__clone()方法对一个对象实例进行的浅复制,对象内的基本数值类型进行的是传值复制，而对象内的对象型成员变量,如果不重写__clone方法,显式的clone这个对象成员变量的话,这个成员变量就是传引用复制,而不是生成一个新的对象

### __call(string $function_name, array $arguments)
该方法在调用的方法不存在时会自动调用

### is_callable() && method_exists()
-  bool is_callable ( mixed $var [, bool $syntax_only = false [, string &$callable_name ]] )
其中，
参数1 - 可以是一个函数名字符串，也可以是一个数组(第一个参数是类名，第二个是方法名)
参数2 - 如果将该参数设置为true，函数仅仅检查给定的方法或函数名称的语法是否正确，而不检查其是否真正存在
参数3 - 只可能是 someFunction 或者 someClass::someMethod

注意： 类的构造函数(__contrunct 或者 与类名同名的函数)，返回false

- bool method_exists ( object $object , string $method_name )
如果 method_name 所指的方法在 object 所指的对象类中已定义，则返回 TRUE，否则返回 FALSE。

区别 ： 对于 private，protected和public类型的方法，method_exits()会返回true；但是is_callable()会检查存在其是否可以访问，如果是private，protected类型的，它会返回false