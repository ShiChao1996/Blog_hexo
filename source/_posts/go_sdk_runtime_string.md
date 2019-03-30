---
title:  Go String 笔记
cover: https://image.littlechao.top/20180301160118000011.jpg
author: 
  nick: Lovae
date: 2018-6-15
categories: go sdk
tags: 
    - golang
subtitle: Go runtime/string.go 阅读笔记

---
## Go String 笔记

### 什么是 string ？

标准库`builtin`的解释：

```
type string

string is the set of all strings of 8-bit bytes, conventionally but not necessarily representing UTF-8-encoded text. A string may be empty, but not nil. Values of string type are immutable.
```

简单的来说字符串是一系列 8 位字节的集合，通常但不一定代表 UTF-8 编码的文本。字符串可以为空，但不能为  nil。而且字符串的值是不能改变的。
不同的语言字符串有不同的实现，在 go 的源码中 `src/runtime/string.go`，string 在底层的定义如下：

```go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

可以看到 str 其实是个指针，指向某个数组的首地址，这个数组就是一个字节数组，里面存着 string 的真正内容。其实字节数组指针更像是 c 语言的字符串形式，而在 go 里，对其进行封装。不同的是， c 语言的 string 是以 null 或 `/0` 结尾，计算长度的时候对其遍历；而 go 的 string 结尾没有特殊符号，只不过用空间换时间，把长度存在了 len 字段里。

那么问题来了，我们平时用的 string 又是什么呢？它的定义如下：

```go
type string string
```

。。。好像和刚刚说的不太一样哈(-_-!)。这个 string 就是一个*名叫 string 的类型*，其实什么也不代表。只不过为了直观，使用的时候，把 stringStruct 转换成 string 类型。

```Go
func gostringnocopy(str *byte) string {
	ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
	s := *(*string)(unsafe.Pointer(&ss))
	return s
}
```

为了验证，我们可以试一下：

```go
package main

import (
   "fmt"
   "unsafe"
)

func main()  {
   var a = "nnnn"
   fmt.Println(a)
   var b = (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&a)) + 8));
    // 按照 stringStruct 结构，把 a 地址偏移 int 的长度位，得到 len 字段地址
    // 这里我的电脑是 64 位，而系统寻址以一个在节为单位，所以 +8
   fmt.Println(*b) // 这里输出的是 a 的长度 4
}
```

### string 操作

#### 拼接

我们可以用 + 来完成字符串的拼接，就像这样：s := x+y+z+… 底层如何实现的呢？

```go
type tmpBuf [tmpStringBufSize]byte // 这是一个很重要的类型，tmpStringBufSize 为常量 32，但这个值并没有什么科学依据(-_-!)

func concatstrings(buf *tmpBuf, a []string) string {// 把所有要拼接的字符串放到 a 里面
	idx := 0
	l := 0
	count := 0
	for i, x := range a { // 这里主要计算总共需要的长度，以便分配内存
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
        // 需要注意的是，虽然空字符串看起来不占空间，可是底层还是 stringStruct，仍要占两个 int 空间
	}

	if count == 1 && (buf != nil || !stringDataOnStack(a[idx])) {
		return a[idx] // count 为 1 表明不需要拼接，直接返回源 string，并且没有内存拷贝
	}
	s, b := rawstringtmp(buf, l) // 这里分配了一个长度为 l 字节的内存，这个内存并没有初始化
	for _, x := range a {
		copy(b, x) // 把每个字符串的内容复制到新的字节数组里面
		b = b[len(x):]
	}
	return s
}
```

可是这里有个问题，b 是一个字节切片，而 x 是字符串，为什么能直接复制呢？

#### 与切片的转换

内置函数copy会有一种特殊情况`copy(dst []byte, src string) int`，但是两者并不能直接 copy，需要把 string 转换成 []byte。

```go
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{} // 清零
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}

// 申请新的内存，返回切片
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size)) // 使申请内存的大小为 8 的倍数
	p := mallocgc(cap, nil, false) // 第三个参数为 FALSE 表示不用给分配的内存清零
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size)) // 超出需要的部分内存清零
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```

### string 与内存

##### string 字面量

前面提到过，字符串的值是不能改变的，可是为什么呢？

这里说的字符串通常指的是 *字符串字面量*，因为它的存储位置不在堆和栈上，通常 string 常量是编译器分配到**只读段**的(.rodata)，对应的数据地址不可修改。

不过等等，好像有什么不对？下面的代码为啥改了呢？

```go
var str = "aaaa"
str = "bbbb"
```

这是因为前面提到过的 stringStruct，我们拿到的 str 实际上是 stringStruct 转换成 string 的。常量`aaaa`被保存在了只读段，下面函数参数 str 就是这个常量的地址：

```Go
func gostringnocopy(str *byte) string {
	ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
	s := *(*string)(unsafe.Pointer(&ss))
	return s
}
```

所以我们拿到的 str 本来是 stringStruct.str ，给 str 赋值相当于给 stringStruct.str 赋值，使其指向 `bbbb`所在地只读段地址，而 `aaaa`本身是没有改变的。在改变 stringStruct.str 的同时，解释器也会更新 stringStruct.len 的值。

##### 动态 string

所谓动态是指字符串 stringStruct.str 指向的地址不在只读段，而是指向由 malloc 动态分配的堆地址。尽管如此，直接修改 string 的内容还是非法的。要修改内容，可以先把 string 转成 []byte，不过这里会有一次内存拷贝，这点在转换的代码中可以看到。不过也可以做到 ‘零拷贝转换’：

```go
func stringtoslicebyte(s string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	bh := reflect.SliceHeader{
		Data: sh.Data,
		Len:  sh.Len,
		Cap:  sh.Len,
	}
	return *(*[]byte)(unsafe.Pointer(&bh))
}
```

不过这种方法不建议使用，因为一旦 string 指向的内存位于只读段，转换成 []byte 后对其进行写操作会引发系统的段错误。

##### 临时 string

有时候我们会把 []byte 转换成 string，通常也会发生一次内存拷贝，但有的时候我们只需要 ‘临时的’ 字符串，比如：

*  使用 m[string(k)] 来查找map
* 用作字符拼接： `"<"+string(b)+">"`
* 用于比较： string(b)=="foo"

这些情况下我们都只是临时的使用一下一个 []byte 的字符串形式的值，如果分配内存有点不划算，所以编译器会做出一些优化，使用如下函数来转换：

```go
func slicebytetostringtmp(b []byte) string {
	if raceenabled && len(b) > 0 {
		racereadrangepc(unsafe.Pointer(&b[0]),
			uintptr(len(b)),
			getcallerpc(),
			funcPC(slicebytetostringtmp))
	}
	if msanenabled && len(b) > 0 {
		msanread(unsafe.Pointer(&b[0]), uintptr(len(b)))
	}
    // 注意，以上两个 if 都为假，所以不会执行。不知道有什么用(-_-!)
	return *(*string)(unsafe.Pointer(&b))
}
```



---

以上是读了 `src/runtime/string.go` 代码的一些个人想法，连蒙带猜，所以有些地方可能不太对，欢迎指出啦(^_^)！