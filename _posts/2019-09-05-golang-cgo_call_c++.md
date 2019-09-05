---
layout:     post
title:      Golang cgo调用c++动态库so文件
category: golang
tags: [golang]
description: 我们知道，go只能通过cgo调用c库，而不能直接调用c++库；下面介绍cgo怎么调用c++动态库
---

如果你有一个c++做的动态链接库.so文件,而你只有一些相关类的声明,那么你如何用c调用呢,

C++创始人在编写C++的时候，C语言正盛行，他不得不让C++兼容C。C++最大的特性就是封装，继承，多态，重载。而这些特性恰恰是C语言所不具备的。至于多态，核心技术是通过虚函数表实现的，其实也就是指针。而对于重载，与C语言相比，其实就是编译方式不同而已： C++编译方式和C编译方式。对于函数调用，编译器只要知道函数的参数类型和返回值以及函数名就可以进行编译连接。那么为了让C调用C++接口或者是说C++调用C接口，就必须是调用者和被调用者有着同样的编译方式。这既是extern "C"的作用，extern “C”是的程序按照C的方式编译。


## C++代码

/var/www/cgo_test/student.h

```
#include <iostream>
using namespace std;

class Student {
public:
    Student(){}
    ~Student(){}
    void Operation();
    void SetName(string name);
    string name;
};
```

/var/www/cgo_test/student.cpp

```
using namespace std;
void Student::Operation()
{
    cout << "Hi my name is " << name <<endl;
}
void Student::SetName(string name1)
{
        name = name1;
}
```

/var/www/cgo_test/interface.h

```
#ifdef __cplusplus
extern "C"{
#endif
 
void *stuCreate();
void initName(void *, char* name);
void getStuName(void *);
void getName();
 
#ifdef __cplusplus
}
#endif
```

/var/www/cgo_test/interface.cpp

```
#include "student.h"
#include "interface.h"
 
#ifdef __cplusplus
extern "C"{
#endif
 
void *stuCreate()
{
        return new Student(); //构造
}

void getStuName(void *p)
{
    static_cast<Student *>(p)->Operation();
}

void initName(void *p, char* name1)
{
    static_cast<Student *>(p)->SetName(name1);
}

void getName()
{
        Student obj;
        obj.Operation();
}
 
#ifdef __cplusplus
}
#endif
```


## C代码

```
#include "interface.h"
 
int main()
{
    void *p = stuCreate();
    char *name = "test";
    initName(p, name);
    getStuName(p);
    getName();
    return 0;
}
```

## 编译流程

1. 首先生成动态库

```
g++ student.cpp interface.cpp -fPIC -shared -o libstu.so
```

2. 编译main.c

```
gcc main.c -L.  -lstu
```

3. 运行./a.out
./a.out

到这里，c已经成功调用c++代码

执行结果

> Hi my name is test
> Hi my name is


其中，上面步骤2时可以会报错

> error while loading shared libraries: libstu.so: cannot open shared object file: No such file or directory

解决方法 [more](http://www.blogjava.net/zhyiwww/archive/2007/12/14/167827.html)

1. 增加配置路径

vim /etc/ld.so.conf.d/test.conf

```
/var/www/cgo_test
```

2. /sbin/ldconfig
3. ldd a.out 查看已经可以成功链接到 libstu.so


## go代码

/var/www/cgo_test/test.go

```
// +build linux
// +build amd64
// +build !noptlogin

package  main

/*
#cgo CFLAGS: -I./
#cgo LDFLAGS: -L./ -lstu 
#include <stdlib.h>
#include <stdio.h>
#include "interface.h" //非标准c头文件，所以用引号
*/
import "C"

import(
    "unsafe"
)

func main() {
    name := "test!"
    cStr := C.CString(name)
    defer C.free(unsafe.Pointer(cStr))
    obj := C.stuCreate()
    C.initName(obj, cStr)
    C.getStuName(obj)
}
```

然后直接执行 ```go run test.go``` 时会报错

> package main: build constraints exclude all Go files in /var/www/cgo_test

> /usr/bin/ld: skipping incompatible ./lib/libstu.so when searching for -lstu

> /usr/bin/ld: cannot find -lstu

> collect2: error: ld returned 1 exit status

解决方法

go可能默认不支持cgo，```go env``` 可查看相关配置，可通过显示指定 ```CGO_ENABLED=1```、```GOARCH=amd64``` 解决

```
 GOARCH=amd64 GOOS=linux CGO_ENABLED=1 go ru n test.go
 ```
 
 执行结果
 
 > Hi my name is test!


##  参考资料

1. [深入Golang之CGO（原理）](http://www.opscoder.info/golang_cgo.html)
2. [在Go函数中调用c动态库](https://studygolang.com/articles/10163)
3. [用g++编译生成动态连接库*.so的方法及连接](https://blog.csdn.net/ACb0y/article/details/6553051)
4. [linux找不到.so文件的解决方法](http://www.blogjava.net/zhyiwww/archive/2007/12/14/167827.html)
5. [Linux C语言调用C++动态链接库](https://blog.csdn.net/sjin_1314/article/details/20958149)
6. [C 调用 C++ 类](https://blog.csdn.net/fengfengdiandia/article/details/82704375)
