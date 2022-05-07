---
layout:     post
title:      字符串拼接的几种方式对比
category:   [golang]
tags:       [string]
description: golang中，我们常用的字符串拼接方式有五种，它们的性能和内存使用情况如何呢？
---

golang 中字符串拼接常用的方式有 5 种

- 运算符加号 ➕
- fmt.Sprintf
- strings.Join
- bytes.Buffer
- strings.Builder

那这些拼接方式中，效率又是怎样的呢？我们应该用哪个比较好？

## benchmark

基准测试是测量一个程序在固定工作负载下的性能，Go 语言也提供了可以支持基准性能测试的 benchmark。

### 使用方法

下面展示一个基准测试的示例代码来剖析下它的使用方式：

```
func Benchmark_test(b *testing.B) {
	for i := 0; i < b.N ; i++ {
		s := make([]int, 0)
		for i := 0; i < 10000; i++ {
			s = append(s, i)
		}
	}
}
```

- 进行基准测试的文件必须以\*\_test.go 的文件为结尾，这个和测试文件的名称后缀是一样的，例如 abc_test.go
- 参与 Benchmark 基准性能测试的方法必须以 Benchmark 为前缀，例如 BenchmarkABC()
- 参与基准测试函数必须接受一个指向 Benchmark 类型的指针作为唯一参数，\*testing.B
- 基准测试函数不能有返回值
- b.ResetTimer 是重置计时器，调用时表示重新开始计时，可以忽略测试函数中的一些准备工作
- b.N 是基准测试框架提供的，表示循环的次数，因为需要反复调用测试的代码，才可以评估性能

### 命令及参数

性能测试命令为 go test [参数]，比如 go test -bench=. -benchmem，具体的命令参数及含义如下：

| 参数          | 含义                                                                    |
| ------------- | ----------------------------------------------------------------------- |
| -bench regexp | 性能测试，支持表达式对测试函数进行筛选。                                |
| -bench .      | 则是对所有的 benchmark 函数测试，指定名称则只执行具体测试方法而不是全部 |
| -benchmem     | 性能测试的时候显示测试函数的内存分配的统计信息                          |
| －count n     | 运行测试和性能多少此，默认一次                                          |
| -run regexp   | 只运行特定的测试函数， 比如-run ABC 只测试函数名中包含 ABC 的测试函数   |
| -timeout t    | 测试时间如果超过 t, panic,默认 10 分钟                                  |
| -v            | 显示测试的详细信息，也会把 Log、Logf 方法的日志显示出来                 |

### 测试结果

执行命令后，性能测试的结果展示如下

```
$ go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: program/benchmark
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
Benchmark_test-12        7439091               152.0 ns/op           248 B/op          5 allocs/op
PASS
ok      promgram/benchmark    1.304s
```

对以上结果进行逐一分析：

| 结果项            | 含义                                   |
| ----------------- | -------------------------------------- |
| Benchmark_test-12 | Benchmark_test 是测试的函数名          |
| -12               | 表示 GOMAXPROCS（线程数）的值为 12     |
| 7439091           | 表示一共执行了 7439091 次，即 b.N 的值 |
| 152.0 ns/op       | 表示平均每次操作花费了 152.0 纳秒      |
| 248B/op           | 表示每次操作申请了 248Byte 的内存申请  |
| 5 allocs/op       | 表示每次操作申请了 5 次内存            |

## 字符串拼接的几种方式

### 操作符号 ➕

常用的方式

test.go

```
package main

import "fmt"

func main() {
	name := "name"
	val := "tony"
	age := "18"
	sex := "male"
	country := "china"
	city := "bj"
	str := name + val + sex + age + country + city
	fmt.Println(str)
}
```

通过 go tool compile -S -N -l test.go 命令查看调用方式

```
        0x021c 00540 (test.go:12)       MOVL    $6, CX
        0x0221 00545 (test.go:12)       MOVQ    CX, DI
        0x0224 00548 (test.go:12)       PCDATA  $1, $0
        0x0224 00548 (test.go:12)       CALL    runtime.concatstrings(SB)
        0x0229 00553 (test.go:12)       MOVQ    AX, "".str+88(SP)
        0x022e 00558 (test.go:12)       MOVQ    BX, "".str+96(SP)
        0x0233 00563 (test.go:13)       MOVUPS  X15, ""..autotmp_8+168(SP)
        0x023c 00572 (test.go:13)       LEAQ    ""..autotmp_8+168(SP), DX
        0x0244 00580 (test.go:13)       MOVQ    DX, ""..autotmp_12+40(SP)
```

我们看到，通过 ➕ 对字符串拼接时，我们实际上调用了 go 的 runtime 包里的函数 runtime.concatstrings 函数

```
func concatstrings(buf *tmpBuf, a []string) string {
	idx := 0
	l := 0
	count := 0
	for i, x := range a {
		n := len(x)
		if n == 0 {
			continue
		}
		if l+n < l {
			throw("string concatenation too long")
		}
		l += n
		count++
		idx = i
	}
	if count == 0 {
		return ""
	}

	// If there is just one string and either it is not on the stack
	// or our result does not escape the calling frame (buf != nil),
	// then we can return that string directly.
	if count == 1 && (buf != nil || !stringDataOnStack(a[idx])) {
		return a[idx]
	}
	//会涉及内存申请
	s, b := rawstringtmp(buf, l)
	for _, x := range a {
		//内存拷贝
		copy(b, x)
		b = b[len(x):]
	}
	return s
}
```

在每次调用时，会申请一次内存用于保存拼接后的结果，然后把每个字符串拷贝进去，最后返回内存结果

既然每次调用都会进行一次内存分配，那么对于 N 多个字符串拼接，我们在一个表达式中拼接完成和通过多个表达式进行拼接，结果是否一样呢？

上面的 demo 就是在一个表达式中拼接完成的，通过多个表达式拼接如下所示

```
str := name + val
str += sex
str += age
str += country
str += city
```

这两种方式性能和内存消耗情况如何呢？我们会在后面稍后分析

### fmt.Sprintf

Sprintf 函数的实现非常简单

```
func Sprintf(format string, a ...interface{}) string {
	p := newPrinter()
	p.doPrintf(format, a)
	s := string(p.buf)
	p.free()
	return s
}
```

doPrintf 是核心的打印方法

```
func (p *pp) doPrintf(format string, a []interface{}) {
	end := len(format)
	argNum := 0         // we process one argument per non-trivial format
	afterIndex := false // previous item in format was an index like [3].
	p.reordered = false
formatLoop:
	for i := 0; i < end; {
		p.goodArgNum = true
		lasti := i
		for i < end && format[i] != '%' {
			i++ // 一直累加 直到遇到 ‘%’ 或者到达format的末尾
		}
		if i > lasti {
			p.buf.WriteString(format[lasti:i]) // 输出普通字符
		}
		if i >= end {
			// done processing format string
			break
		}

		// Process one verb
		i++ // 将索引后移 比如遇到了%d 那么i++之后就会指向d

		// Do we have flags?
		p.fmt.clearflags() // 确保fmtFlags被清空 可能并不需要这么做 但是加上比较稳妥
	simpleFormat:
		for ; i < end; i++ {
			c := format[i]
            // 记录 '#' '0' '+' '-' ' ' 标记 在打印不同类型的地方 会根据标记 有不同的处理
			switch c {
			case '#':
				p.fmt.sharp = true
			case '0':
				p.fmt.zero = !p.fmt.minus // Only allow zero padding to the left.
			case '+':
				p.fmt.plus = true
			case '-':
				p.fmt.minus = true
				p.fmt.zero = false // Do not pad with zeros to the right.
			case ' ':
				p.fmt.space = true
			default:
				// Fast path for common case of ascii lower case simple verbs
				// without precision or width or argument indices.
				if 'a' <= c && c <= 'z' && argNum < len(a) {
					// %v 打印值的默认格式
					if c == 'v' {
						// %+v 如果是结构体 打印字段名
						// Go syntax
						p.fmt.sharpV = p.fmt.sharp
						p.fmt.sharp = false
						// %#v 用go的语法格式打印
						// Struct-field syntax
						p.fmt.plusV = p.fmt.plus
						p.fmt.plus = false
					}
					// 打印参数
					p.printArg(a[argNum], rune(c))
					argNum++
					i++
					continue formatLoop
				}
				// Format is more complex than simple flags and a verb or is malformed.
				break simpleFormat
			}
		}

		// 处理使用[] 指定参数值索引的情况
		// Do we have an explicit argument index?
		argNum, i, afterIndex = p.argNumber(argNum, format, i, len(a))

		// ‘*’ 通过参数指定宽度或精度
		// Do we have width?
		if i < end && format[i] == '*' {
			i++
			p.fmt.wid, p.fmt.widPresent, argNum = intFromArg(a, argNum)

			if !p.fmt.widPresent {
				p.buf.WriteString(badWidthString)
			}

			// We have a negative width, so take its value and ensure
			// that the minus flag is set
			if p.fmt.wid < 0 {
				p.fmt.wid = -p.fmt.wid
				p.fmt.minus = true
				p.fmt.zero = false // Do not pad with zeros to the right.
			}
			afterIndex = false
		} else {
			// 使用数字指定宽度
			p.fmt.wid, p.fmt.widPresent, i = parsenum(format, i, end)
			if afterIndex && p.fmt.widPresent { // "%[3]2d"
				p.goodArgNum = false
			}
        }

		// 处理精度
		// Do we have precision?
		if i+1 < end && format[i] == '.' {
			i++
			if afterIndex { // "%[3].2d"
				p.goodArgNum = false
			}
			argNum, i, afterIndex = p.argNumber(argNum, format, i, len(a))
			if i < end && format[i] == '*' {
				i++
				p.fmt.prec, p.fmt.precPresent, argNum = intFromArg(a, argNum)
				// Negative precision arguments don't make sense
				if p.fmt.prec < 0 {
					p.fmt.prec = 0
					p.fmt.precPresent = false
				}
				if !p.fmt.precPresent {
					p.buf.WriteString(badPrecString)
				}
				afterIndex = false
			} else {
				p.fmt.prec, p.fmt.precPresent, i = parsenum(format, i, end)
				if !p.fmt.precPresent {
					p.fmt.prec = 0
					p.fmt.precPresent = true
				}
			}
		}

		if !afterIndex {
			argNum, i, afterIndex = p.argNumber(argNum, format, i, len(a))
		}

		if i >= end {
			p.buf.WriteString(noVerbString)
			break
		}

		verb, size := rune(format[i]), 1
		if verb >= utf8.RuneSelf {
			verb, size = utf8.DecodeRuneInString(format[i:])
		}
		i += size

		switch {
		case verb == '%': // Percent does not absorb operands and ignores f.wid and f.prec.
			p.buf.WriteByte('%') // 使用 ‘%%’ 来打印 ‘%’
		case !p.goodArgNum:
			p.badArgNum(verb)
		case argNum >= len(a): // No argument left over to print for the current verb.
			p.missingArg(verb) // 参数数量不足
		case verb == 'v':
			// Go syntax
			p.fmt.sharpV = p.fmt.sharp
			p.fmt.sharp = false
			// Struct-field syntax
			p.fmt.plusV = p.fmt.plus
			p.fmt.plus = false
			fallthrough
		default:
			p.printArg(a[argNum], verb)
			argNum++
		}
	}

	// 处理参数多于占位符的情况
	// Check for extra arguments unless the call accessed the arguments
	// out of order, in which case it's too expensive to detect if they've all
	// been used and arguably OK if they're not.
	if !p.reordered && argNum < len(a) {
		p.fmt.clearflags()
		p.buf.WriteString(extraString)
		for i, arg := range a[argNum:] {
			if i > 0 {
				p.buf.WriteString(commaSpaceString)
			}
			if arg == nil {
				p.buf.WriteString(nilAngleString)
			} else {
				p.buf.WriteString(reflect.TypeOf(arg).String())
				p.buf.WriteByte('=')
				p.printArg(arg, 'v')
			}
		}
		p.buf.WriteByte(')')
	}
}
```

printArg
printArg 接收两个参数
arg：参数值
verb：占位符类型
方法中通过 switch 参数的 type，分别处理不同的参数类型。

```
func (p *pp) printArg(arg interface{}, verb rune) {
	p.arg = arg
	p.value = reflect.Value{}

	if arg == nil {
		switch verb {
		case 'T', 'v':
			p.fmt.padString(nilAngleString)
		default:
			p.badVerb(verb)
		}
		return
	}

	// Special processing considerations.
	// %T (the value's type) and %p (its address) are special; we always do them first.
	switch verb {
	case 'T': // 打印类型
		p.fmt.fmt_s(reflect.TypeOf(arg).String())
		return
	case 'p': // 打印指针
		p.fmtPointer(reflect.ValueOf(arg), 'p')
		return
	}

	// Some types can be done without reflection.
	switch f := arg.(type) {
	case bool:
		p.fmtBool(f, verb)
	case float32:
		p.fmtFloat(float64(f), 32, verb)
	case float64:
		p.fmtFloat(f, 64, verb)
	case complex64:
		p.fmtComplex(complex128(f), 64, verb)
	case complex128:
		p.fmtComplex(f, 128, verb)
	case int:
		p.fmtInteger(uint64(f), signed, verb)
	case int8:
		p.fmtInteger(uint64(f), signed, verb)
	case int16:
		p.fmtInteger(uint64(f), signed, verb)
	case int32:
		p.fmtInteger(uint64(f), signed, verb)
	case int64:
		p.fmtInteger(uint64(f), signed, verb)
	case uint:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint8:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint16:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint32:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint64:
		p.fmtInteger(f, unsigned, verb)
	case uintptr:
		p.fmtInteger(uint64(f), unsigned, verb)
	case string:
		p.fmtString(f, verb)
	case []byte:
		p.fmtBytes(f, verb, "[]byte")
	case reflect.Value:
		// Handle extractable values with special methods
		// since printValue does not handle them at depth 0.
		if f.IsValid() && f.CanInterface() {
			p.arg = f.Interface()
			if p.handleMethods(verb) {
				return
			}
		}
		p.printValue(f, verb, 0)
	default:
		// If the type is not simple, it might have methods.
		if !p.handleMethods(verb) {
			// Need to use reflection, since the type had no
			// interface methods that could be used for formatting.
			p.printValue(reflect.ValueOf(f), verb, 0)
		}
	}
}
```

我们看到，在 Sprintf 实现过程中，用到了 interface、reflect。因为反射而损失了部分性能，因此在性能关键的地方应该减少应用。

### bytes.Buffer

bytes.buffer 是一个缓冲 byte 类型的缓冲器存放着都是 byte
是一个变长的 buffer，具有 Read 和 Write 方法。 Buffer 的 零值 是一个 空的 buffer，但是可以使用，底层就是一个 []byte， 字节切片

常用实现方式

```
		var buffer bytes.Buffer
		n := 0
		for j := 0; j < len(list); j++ {
			n += len(list[j])
		}
		buffer.Grow(n)
		for j := 0; j < len(list); j++ {
			buffer.WriteString(list[j])
		}
		tmp := buffer.String()
		_ = tmp
```

### strings.Builder

常用实现方式

```
		var builder strings.Builder
		for j := 0; j < len(list); j++ {
			builder.WriteString(list[j])
		}
		tmp := builder.String()
		_ = tmp
```

strings.Builder 的底层结构&实现方式和 bytes.Buffer 相差不多。主要不同的是 String()方法与 Buffer 的 string 方法有明显区别。Buffer 的 string 是一种强转，我们知道在强转的时候是需要进行申请空间，并拷贝的。而 Builder 只是指针的转换。

所以 Buffer 的 string 会多进行一次内存分配

### strings.Join

```
func Join(elems []string, sep string) string {
	switch len(elems) {
	case 0:
		return ""
	case 1:
		return elems[0]
	}
	n := len(sep) * (len(elems) - 1)
	for i := 0; i < len(elems); i++ {
		n += len(elems[i])
	}

	var b Builder
	b.Grow(n)
	b.WriteString(elems[0])
	for _, s := range elems[1:] {
		b.WriteString(sep)
		b.WriteString(s)
	}
	return b.String()
}
```

可以看到 strings.Join 其实就是对 strings.Builder 的一种实现，只是多了一个拼接符

## 性能对比

我们主要对比了拼接 2 个、10 个、100 个字符串，这五种方式的效率与内存消耗情况

string_test.go

```
package string_test10_test

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
)

var (
	list = []string{"test1", "test2", "test3", "test4", "test5", "test6", "test7", "test8", "test9", "test10"}
)

func BenchmarkOperatorOnce(b *testing.B) {
	for i := 0; i < b.N; i++ {
		tmp := list[0] + list[1] + list[2] + list[3] + list[4] + list[5] + list[6] + list[7] + list[8] + list[9]
		_ = tmp
	}
}

func BenchmarkOperatorLoop(b *testing.B) {
	for i := 0; i < b.N; i++ {
		tmp := ""
		for _, data := range list {
			tmp += data
		}
		_ = tmp
	}
}

func BenchmarkSprintf(b *testing.B) {
	for i := 0; i < b.N; i++ {
		tmp := fmt.Sprintf("%s%s%s%s%s%s%s%s%s%s", list[0], list[1], list[2], list[3], list[4], list[5], list[6], list[7], list[8], list[9])
		_ = tmp
	}
}

func BenchmarkJoin(b *testing.B) {
	for i := 0; i < b.N; i++ {
		tmp := strings.Join(list, "")
		_ = tmp
	}
}

func BenchmarkBuffer(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var buffer bytes.Buffer
		for j := 0; j < len(list); j++ {
			buffer.WriteString(list[j])
		}
		tmp := buffer.String()
		_ = tmp
	}
}

func BenchmarkBufferOptimize(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var buffer bytes.Buffer
		n := 0
		for j := 0; j < len(list); j++ {
			n += len(list[j])
		}
		buffer.Grow(n)
		for j := 0; j < len(list); j++ {
			buffer.WriteString(list[j])
		}
		tmp := buffer.String()
		_ = tmp
	}
}

func BenchmarkBuilder(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var builder strings.Builder
		for j := 0; j < len(list); j++ {
			builder.WriteString(list[j])
		}
		tmp := builder.String()
		_ = tmp
	}
}

func BenchmarkBuilderOptimize(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var builder strings.Builder
		n := 0
		for j := 0; j < len(list); j++ {
			n += len(list[j])
		}
		builder.Grow(n)
		for j := 0; j < len(list); j++ {
			builder.WriteString(list[j])
		}
		tmp := builder.String()
		_ = tmp
	}
}

```

其中：

- OperatorOnce：是用加号 ➕ 进行拼接的，它是在一个表达式中完成的
- OperatorLoop：也是用加号 ➕ 进行拼接的，不同的是它通过循环方式在多个表达式中完成
- Sprintf：就是常规的 Sprintf 实现
- Buffer：常规的 bytes.Buffer 实现，没有预分配内存，后面可能会涉及到扩容，重新申请内存
- BufferOptimize：对常规的优化，在申请内存时一次申请足够内存，后面不会进行扩容
- Builder：常规的 strings.Builder 实现，没有预分配内存，后面可能会涉及到扩容，重新申请内存
- BuilderOptimize：对常规的优化，在申请内存时一次申请足够内存，后面不会进行扩容

我们通过 go test -benchmem -bench=. 命令对上面三种规则分别进行测试

字符串个数=100

| 运行函数                    | 运行次数 | 单次运行耗时 | 内存使用情况 | 内存分配次数  |
| --------------------------- | -------- | ------------ | ------------ | ------------- |
| BenchmarkOperatorOnce-16    | 1772316  | 707.6 ns/op  | 640 B/op     | 1 allocs/op   |
| BenchmarkOperatorLoop-16    | 126884   | 8531 ns/op   | 30952 B/op   | 99 allocs/op  |
| BenchmarkSprintf-16         | 269764   | 4240 ns/op   | 2241 B/op    | 101 allocs/op |
| BenchmarkJoin-16            | 1789785  | 678.0 ns/op  | 640 B/op     | 1 allocs/op   |
| BenchmarkBuffer-16          | 1221356  | 995.9 ns/op  | 2864 B/op    | 6 allocs/op   |
| BenchmarkBufferOptimize-16  | 1686231  | 707.4 ns/op  | 1280 B/op    | 2 allocs/op   |
| BenchmarkBuilder-16         | 1595317  | 763.6 ns/op  | 1912 B/op    | 8 allocs/op   |
| BenchmarkBuilderOptimize-16 | 2549725  | 473.0 ns/op  | 640 B/op     | 1 allocs/op   |

字符串个数=10

| 运行函数                    | 运行次数 | 单次运行耗时 | 内存使用情况 | 内存分配次数 |
| --------------------------- | -------- | ------------ | ------------ | ------------ |
| BenchmarkOperatorOnce-16    | 14374328 | 75.58 ns/op  | 64 B/op      | 1 allocs/op  |
| BenchmarkOperatorLoop-16    | 3810916  | 322.3 ns/op  | 328 B/op     | 9 allocs/op  |
| BenchmarkSprintf-16         | 2776459  | 459.7 ns/op  | 224 B/op     | 11 allocs/op |
| BenchmarkJoin-16            | 12845788 | 83.82 ns/op  | 64 B/op      | 1 allocs/op  |
| BenchmarkBuffer-16          | 11171432 | 102.0 ns/op  | 128 B/op     | 2 allocs/op  |
| BenchmarkBufferOptimize-16  | 11147169 | 111.2 ns/op  | 128 B/op     | 2 allocs/op  |
| BenchmarkBuilder-16         | 8650586  | 128.9 ns/op  | 120 B/op     | 4 allocs/op  |
| BenchmarkBuilderOptimize-16 | 18879309 | 65.36 ns/op  | 64 B/op      | 1 allocs/op  |

字符串个数=2

| 运行函数                    | 运行次数 | 单次运行耗时 | 内存使用情况 | 内存分配次数 |
| --------------------------- | -------- | ------------ | ------------ | ------------ |
| BenchmarkOperatorOnce-16    | 71627631 | 14.86 ns/op  | 0 B/op       | 0 allocs/op  |
| BenchmarkOperatorLoop-16    | 29431378 | 39.33 ns/op  | 16 B/op      | 1 allocs/op  |
| BenchmarkSprintf-16         | 9684672  | 126.7 ns/op  | 48 B/op      | 3 allocs/op  |
| BenchmarkJoin-16            | 35210997 | 31.27 ns/op  | 16 B/op      | 1 allocs/op  |
| BenchmarkBuffer-16          | 28145877 | 43.49 ns/op  | 64 B/op      | 1 allocs/op  |
| BenchmarkBufferOptimize-16  | 22528340 | 47.48 ns/op  | 64 B/op      | 1 allocs/op  |
| BenchmarkBuilder-16         | 25559436 | 46.91 ns/op  | 24 B/op      | 2 allocs/op  |
| BenchmarkBuilderOptimize-16 | 36287246 | 28.56 ns/op  | 16 B/op      | 1 allocs/op  |

### 结论

从上面三组结果可以看到

1. 在字符串个数比较少时，操作符号 ➕ 性能也是很好的，不过需要在同一个表达式中完成；只是我们在平时业务开发中，如果需要拼接的字符串个数比较多且不固定时，在同一个表达式中拼接不太好实现，不过也不要需要循环进行多次拼接，性能可能会很差
2. 当字符串个数比较多且不固定时，用 strings.Builder 也是一种很好的选择，如果在进行内存分配时就预分配足够的内存，性能会更好；Buffer 和 Builder 在没有预分配足够内存时，后面发生了扩容，所以有多次内存分配
3. bytes.Buffer 实现方式和 strings.Builder 类似，只是在 string 转成字符串时，它会再分配内存一次，性能比 builder 稍差一些
4. fmt.Sprintf 对于多种不同类型的变量拼接成一个字符串使用很方便，但方便的同时牺牲的是执行的性能

所以建议的话

1. 在字符串个数比较少且固定时，优先使用 ➕ 进行拼接
2. 如果字符串个数比较多且不固定时，优先使用 strings.Builder，预先分配足够内存性能会更好；其次 strings.Join 也可以；另外 bytes.Buffer 性能比 builder 稍差一点点，也是一个不错的选择
3. 如果图方便对性能也没有苛刻的要求，比如打日志，也可以使用 fmt.Sprintf 方式

## 参考资料

1. [Benchmark 性能测试](https://blog.csdn.net/u013161278/article/details/117628603)
2. [golang 字符串拼接方式及其性能分析](https://blog.csdn.net/m0_37290103/article/details/116454647)
