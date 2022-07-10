---
layout: post
title: nginx 的rewrite跳转
category: [nginx]
tags: [nginx]
description: 表面看 rewrite 和 location 功能有点像，都能实现跳转，主要区别在于……
---

## 前言

nginx 通过 ngx_http_rewrite_module 模块支持 URI 重写、支持 if 条件判断，但不支持 else。

rewrite 只能放在 server { } 、 location { } 、 if { } 中，并且只能对域名后边的除去传递的参数外的字符串起作用，例如http://aaa.com/a/we/index.php?id=1&u=str只对/a/we/index.php重写。语法为 rewrite regex replacement [flag]

## 指令执行顺序

表面看 rewrite 和 location 功能有点像，都能实现跳转，主要区别在于 rewrite 是在同一域名内更改获取资源的路径，而 location 是对一类路径做控制访问或反向代理，可以 proxy_pass 到其他机器。很多情况下 rewrite 也会写在 location 里，它们的执行顺序是：

1. 执行 server 块的 rewrite 指令（这里的块指的是 server 关键字后{}包围的区域，其它 xx 块类似）
2. 执行location匹配
3. 执行选定的location中的rewrite指令

如果其中某步 URI 被重写，则重新循环执行1-3，直到找到真实存在的文件；
如果循环超过 10 次，则返回 500 Internal Server Error 错误

## 指令详解

### if 指令

- **语法**：if(condition) {…}
- **作用域**： server、location
- **功能**：对给定的条件 condition 进行判断。如果为真，大括号内的 rewrite 指令将被执行。if 条件 (conditon) 可以是如下任何内容

#### if 中的几种判断条件

- 一个变量名；false 如果这个变量是空字符串或者以0开始的字符串；
- 使用 =, != 比较的一个变量和字符串
- 使用 ~， ~* 与正则表达式匹配的变量，如果这个正则表达式中包含}，;则整个表达式需要用” 或’ 包围
- 使用 -f ，!-f 检查一个文件是否存在
- 使用 -d， !-d 检查一个目录是否存在
- 使用 -e ，!-e 检查一个文件、目录、符号链接是否存在
- 使用 -x ， !-x 检查一个文件是否可执行

示例：

```
set $variable "0"; 
if ($variable) {
    # 不会执行，因为 "0" 为 false
    break;            
}

# 使用变量与正则表达式匹配 没有问题
if ( $http_host ~ "^star\.igrow\.cn$" ) {
    break;            
}

# 字符串与正则表达式匹配 报错
if ( "star" ~ "^star\.igrow\.cn$" ) {
    break;            
}
# 检查文件类的 字符串与变量均可
if ( !-f "/data.log" ) {
    break;            
}

if ( !-f $filename ) {
    break;            
}
```

### return 指令

- **语法**：return code [text]; return code URL; return URL;
- **作用域**：server，location，if
- **功能**：停止处理并将指定的 code 码返回给客户端。 非标准 code 码 444 关闭连接而不发送响应报头。

该指令用于检查一个条件是否符合，如果条件符合，则执行大括号内的语句。If 指令不支持嵌套，不支持多个条件 && 和 || 处理。

从0.8.42版本开始， return 语句可以指定重定向 URI (状态码可以为如下几种 301，302，303，307)

也可以为其他状态码指定响应的文本内容，并且重定向的 URI 和响应的文本可以包含变量。

有一种特殊情况，就是重定向的url可以指定为此服务器本地的 URI，这样的话，nginx 会依据请求的协议 $scheme， server_name_in_redirect 和 port_in_redirect 自动生成完整的 URI。


```
location ~ .*\.(sh|bash)?$
{
   return 403;
}
```

### rewrite 指令

- **语法**：rewrite regex replacement [flag];
- **作用域**：server 、location、if
- 功能：如果一个URI匹配指定的正则表达式regex，URI就按照 replacement 重写

rewrite 按配置文件中出现的顺序执行。可以使用 flag 标志来终止指令的进一步处理。

如果 replacement 以 http:// 、 https:// 或 $ scheme 开始，将不再继续处理，这个重定向将返回给客户端。

示例：第一种情况，重写的字符串带 http://


```
location ^~ /redirect {
    # 当匹配前缀表达式 /redirect/(.*)时 请求将被临时重定向到 http://www.$1.com
    # 相当于 flag 写为 redirect
    rewrite ^/(.*)$ http://www.$1.com;
    return 200 "ok";
}
```

在浏览器中输入 127.0.0.1:8080/redirect/baidu ，则临时重定向到 www.baidu.com 

后面的 return 指令将没有机会执行了。


```
location ^~ /redirect {
    rewrite ^/(.*)$ www.$1.com;
    return 200 "ok";
}
# 发送请求如下
# curl 127.0.0.1:8080/redirect/baidu
# ok

```

此处没有带 http:// 所以只是简单的重写。请求的 URI 由 /test1/baidu 重写为 www.baidu.com 因为会顺序执行 rewrite 指令，所以 下一步执行 return 指令，响应后返回 ok

flag 有四种参数可以选择：

1. **last** 停止处理后续 rewrite 指令集，然后对当前重写的新 URI 在 rewrite 指令集上重新查找。
2. **break** 停止处理后续 rewrite 指令集，并不再重新查找，但是当前location 内剩余非 rewrite 语句和 location 外的 非rewrite 语句可以执行。
3. **redirect** 如果 replacement 不是以 http:// 或 https:// 开始，返回 302临时重定向
4. **permanent** 返回 301永久重定向

示例一：


```
# rewrite 后面没有任何 flag 时就顺序执行 
# 当 location 中没有 rewrite 模块指令可被执行时 就重写发起新一轮location 匹配
location / {
    # 顺序执行如下两条rewrite指令 
    rewrite ^/test1 /test2;
    rewrite ^/test2 /test3;  # 此处发起新一轮 location 匹配 URI为/test3
}

location = /test2 {
    return 200 “/test2”;
}  

location = /test3 {
    return 200 “/test3”;
}
# 发送如下请求
# curl 127.0.0.1:8080/test1
# /test3
```

如果正则表达regex式中包含 “}” 或 “;”，那么整个表达式需要用双引号或单引号包围。

示例二：


```
location / {
    rewrite ^/test1 /test2;
    rewrite ^/test2 /test3 last;  # 此处发起新一轮location匹配 uri为/test3
    rewrite ^/test3 /test4;
    proxy_pass http://www.baidu.com;
}

location = /test2 {
    return 200 "/test2";
}  

location = /test3 {
    return 200 "/test3";
}
location = /test4 {
    return 200 "/test4";
}
```

发送如下请求 curl 127.0.0.1:8080/test1 返回 /test3

当如果将上面的 location / 改成如下代码


```
location / {
    rewrite ^/test1 /test2;
    # 此处不会发起新一轮location匹配；当是会终止执行后续rewrite模块指令重写后的 URI 为 /more/index.html
    rewrite ^/test2 /more/index.html break;  
    rewrite /more/index\.html /test4; # 这条指令会被忽略

    # 因为 proxy_pass 不是rewrite模块的指令 所以它不会被 break终止
    proxy_pass https://www.baidu.com;
}
```

发送请求 127.0.0.1:8080/test1
代理到 https://www.baidu.com

#### rewrite 后的请求参数：

如果替换字符串 replacement 包含新的请求参数，则在它们之后附加先前的请求参数。如果你不想要之前的参数，则在替换字符串 replacement 的末尾放置一个问号，避免附加它们。


```
# 由于最后加了个 ?，原来的请求参数将不会被追加到 rewrite 之后的 URI 后面*
rewrite ^/users/(.*)$ /show?user=$1? last;
```

### rewrite_log 指令

- **语法**：rewrite_log on | off;
- **默认值**：rewrite_log off;
- **作用域**：http 、 server 、 location 、 if
- **功能**：开启或关闭以 notice 级别打印 rewrite 处理日志到 error log 文件。

### set指令

- **语法**：set variable value;
- **默认值**：none
- **作用域**：server 、 location 、 if
- **功能**：定义一个变量并赋值，值可以是文本，变量或者文本变量混合体

### uninitialized_variable_warn 指令

- **作用域**：http、 server 、 location 、 if
- **语法**：uninitialized_variable_warn on | off;
- **默认值**：uninitialized_variable_warn on
- **功能**：控制是否记录 有关未初始化变量的警告






