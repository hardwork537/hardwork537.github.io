---
layout:     post
title:      Golang 几种文件路径方式
category: golang
tags: [golang]
description: Golang 中存在各种运行方式，如何正确的引用文件路径成为一个值得商议的问题；下面主要介绍几种常用的文件路径方式
---

## 绝对路径

/home/code/test/go/dir.go

```
package main 

import(
    "fmt"
    "path"
    "runtime"
)

func main() {
    _, filename, _, _ := runtime.Caller(0)
    fmt.Printf("filepath:%s \n", filename)

    confPath := path.Join(filename, "../../conf/conf.ini")
    fmt.Printf("confPath:%s \n", confPath)
}
```

执行结果和当前路径无关，为：

>>>filepath:/home/code/test/go/dir.go 
>>>confPath:/home/code/test/conf/conf.ini


## 相对路径

### 相对当前目录的路径

/home/code/test/go/dir.go

```
package main 

import(
    "fmt"
    "os"
)

func main() {
    filepath, _ := os.Getwd()
    fmt.Printf("curr path:%s \n", filepath)
}
```

如果我在路径 /home/code/test 下执行 go run go/dir.go
执行结果为：
>>> curr path:/home/code/test

而如果我在路径 /home/code/test/go 下执行 go run dir.go
执行结果为：
>>> curr path:/home/code/test/go


### 相对可执行文件路径

/home/code/test/go/dir.go

```
package main 

import(
    "fmt"
    "os"
    "path/filepath"
)

func main() {
    path, _ := filepath.Abs(filepath.Dir(os.Args[0]))
    fmt.Printf("exec path:%s \n", path)
}
```

执行结果和当前路径无关，为：

>>>exec path:/tmp/go-build143529007/b001/exe


## 其他相关实现

```
package utils

import (
	"os"
	"path/filepath"
	"strings"
)

// Get current directory of executable file,
// if fails returns empty filepath and error
func GetCurrentDirectory() (string, error) {
	dir, err := filepath.Abs(filepath.Dir(os.Args[0]))
	if err != nil {
		return "", err
	}
	return dir, nil
}

// returns substring of s, i.e., s[pos:pos+length]
func substr(s string, pos, length int) string {
	runes := []rune(s)
	l := pos + length
	if l > len(runes) {
		l = len(runes)
	}
	return string(runes[pos:l])
}

// Get parental directory of executable file,
// if fails returns empty filepath and error
func GetParentDirectory() (string, error) {
	dirctory, err := GetCurrentDirectory()
	if err != nil {
		return "", err
	}
	parentPath := substr(dirctory, 0, strings.LastIndex(dirctory, string(filepath.Separator)))
	return parentPath, nil
}

// Check whether file is existed,
// if existed returns true, otherwise returns false
func CheckFileIsExist(filename string) bool {
	if _, err := os.Stat(filename); os.IsNotExist(err) {
		return false
	}
	return true
}
```

比如拼接路径可以用

```
import "path"

path.Join(filepath, "conf", "conf.ini")
path.Join(filepath, "conf/conf.ini")
```
