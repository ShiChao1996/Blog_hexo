---
title:  golang unsafe 包
cover: https://image.littlechao.top/20180301160118000011.jpg
author: 
  nick: Lovae
date: 2018-3-4

subtitle: Go unsafe package 理解与注意事项
categories: go sdk
---
## golang unsafe 包

#### ArbitraryType 和 Pointer

Go 语言是强类型语言，并且出于安全的考虑，它不允许不同类型的指针互相转换，比如`*int`不能转为`*float64`。但是它提供了 unsafe 包来做转换。

```go
type ArbitraryType int
type Pointer *ArbitraryType
```

从命名可以看出，ArbitraryType 代表了任意类型，其实，ArbitraryType不是一个真正的类型，它只是一个占位符。而 Pointer 是其指针，并且是一种特殊意义的指针，它可以包含任意类型的地址，有点类似于 C 语言里的`void*` 指针，全能型的。

ArbitraryType 上有三个函数：

```go
func Alignof（variable ArbitraryType）uintptr
func Offsetof（selector ArbitraryType）uintptr
func Sizeof（variable ArbitraryType）uintptr
```

与Golang中的大多数函数不同，上述三个函数的调用将始终在编译时求值，而不是运行时。 这意味着它们的返回结果可以分配给常量。（BTW，unsafe包中的函数中非唯一调用将在编译时求值。当传递给len和cap的参数是一个数组值时，内置函数和cap函数的调用也可以在编译时被求值。）

#### uintptr

`uintptr` 不是 `unsafe` 包的一部分，但是它总是和 `unsafe` 一起用。`uintptr` 是底层内置类型，用于表示指针的值，区别在于`go` 语言中指针不可以参与计算，而 `uintptr` 可以。另外，指针和 uintptr 也是不可以直接转换的。

特别需要注意的是，GC 不会把 uintptr 当成指针，所以由 uintptr 变量表示的地址处的数据也可能被GC回收。

### 用法及注意事项

#### 转换不同类型的指针

```
func Float64bits(f float64) uint64 {
    return *(*uint64)(unsafe.Pointer(&f))
 }
```

#### 把指针转换成 uintptr

```
Converting a Pointer to a uintptr creates an integer value with no pointer semantics
//上面说过的，uintptr 没有指针的含义
```

如下转换：

```go
var a int64 = 0
pa := &a
up := uintptr(unsafe.Pointer(pa))
pa = &int64(1)
```

当 pa 地址改变，uintptr 是不会更新的。且当只有 up 包含了变量 a 的地址，但是 GC 不会把 up 当做指针，所以GC 会回收变量 a 。

#### uintptr 转指针

```go
p := &T{}
p = unsafe.Pointer(uintptr(p) + offset)
```

这里的 offset 得当的话，可以取到 T 类型中没有导出的值，这也是一个巧妙的用法，但是不推荐。注意这里不能写成这样：

```go
p := &T{}	// 1
up := uintptr(p)	// 2
p = unsafe.Pointer(up + offset)	//3
```

这样非常危险，因为有可能在 3 执行之前，up 这个临时变量被 GC ，最终操作的不知道是哪个内存了。因此不能将 uintptr(p) 保存在变量中。

另外，在 C语言 中我们可以将 offset 设成 T 的长度，然后直接对得到的地址进行操作。但是在 go 语言中是不合法的，可以读取，但不应该操作分配给 T 内存之外的部分，会引发 panic：

```go
panic: runtime error: invalid memory address or nil pointer dereference
```

#### 系统调用时转换指针

```go
syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
```

如上，当系统调用需要一个 uintptr 作为参数，也一定把 uintptr(..) 放在系统调用表达式的参数里，以防止被 GC。在系统调用过程中，不必担心 uintptr 失效，它所持有的对象不会被 GC 。

#### reflect.Value.Pointer

在一些函数的返回值中，也可能出现 uintptr，比如 `reflect.Value.Pointer`和`reflect.Value.UnsafeAddr`,对其转换成指针的时候也要注意，不能有中间变量:

```go
p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
//
// As in the cases above, it is invalid to store the result before the conversion:
//
//  INVALID: uintptr cannot be stored in variable
//  before conversion back to Pointer.
//	u := reflect.ValueOf(new(int)).Pointer()
//	p := (*int)(unsafe.Pointer(u))
```

### Summary

- unsafe包用于Go编译器，而不是Go运行时。
- 使用unsafe作为程序包名称只是让你在使用此包是更加小心。
- 使用unsafe.Pointer并不总是一个坏主意，有时我们必须使用它。
- Golang的类型系统是为了安全和效率而设计的。 但是在Go类型系统中，安全性比效率更重要。 通常Go是高效的，但有时安全真的会导致Go程序效率低下。 unsafe包用于有经验的程序员通过安全地绕过Go类型系统的安全性来消除这些低效。
- unsafe包可能被滥用并且是危险的
- 涉及到 uintptr 转指针时，一定注意不能有中间变量

---

#### (续)question

在关于操作不可知内存的时候，会有一些莫名其妙的现象，如下代码是 gocn 上一篇[文章](https://gocn.io/question/371)里的：

```go
func main() {
	illegalUseB()
}

func illegalUseB() {
	a := [4]int{0, 1, 2, 3}
	p := unsafe.Pointer(&a)

	for i := 0; i <= len(a); i++ {
		*(*int)(p) = 1

		fmt.Println(i, ":", *(*int)(p))
		// panic at the above line for the last iteration, when i==4.
		// runtime error: invalid memory address or nil pointer dereference

		p = unsafe.Pointer(uintptr(p) + 8)
	}
}
```

运行这段代码，报错如下：

```go
0 : 1
1 : 1
2 : 1
3 : 1
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x1 pc=0x100ca32]
...
```

但是比较诡异的情况如下：

```go
func illegalUseB() {
	a := [4]int{0, 1, 2, 3}
	p := unsafe.Pointer(&a)

	for i := 0; i <= len(a); i++ {
		fmt.Println(i, ":", *(*int)(p))
		// panic at the above line for the last iteration, when i==4.
		// runtime error: invalid memory address or nil pointer dereference

		p = unsafe.Pointer(uintptr(p) + 8)
        *(*int)(p) = 1	// 调整了位置
	}
}
```

或者如下：

```go
func illegalUseB() {

	a := [4]int{0, 1, 2, 3}
	p := unsafe.Pointer(&a)

	for i := 0; i <= len(a); i++ {
		*(*int)(p) = 1

		fmt.Println(i, ":", *(*int)(p), (*int)(p))	// 多输出了一个值
		// panic at the above line for the last iteration, when i==4.
		// runtime error: invalid memory address or nil pointer dereference

		p = unsafe.Pointer(uintptr(p) + 8)
	}
}
```

这两种情况都不会报错。按理说都是操作了声明变量以外的内存，但是没有向之前一样报错，不知道是什么原因。我的 go SDK 版本是 1.9.1，如果你知道的话麻烦告诉我，谢！