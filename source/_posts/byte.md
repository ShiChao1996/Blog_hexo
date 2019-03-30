---
title: Go bytes.go
cover: https://image.littlechao.top/20180301160118000011.jpg
author: 
  nick: Lovae 
date: 2018-2-5
# 首页每篇文章的子标题
subtitle: Go package bytes.go 阅读笔记
categories: go sdk
tags: 
    - golang
---
> 学习bytes的时候，网上找了很多文章，但基本上是用法，很少对代码的分析。这篇文章对bytes.go 做了简要的分析。

首先从 `import` 的包来看，只引入了`unicode` 包的部分函数，所以这个文件代码主要是对字节切片做字符处理。从主要的功能来分，bytes.go 分为几个部分：
* index/contain 系列函数
* splite、fields 拆分函数
* Map 遍历函数

### prepare
开始对每个部分分析前，先来看一下 `unicode` 包里用的比较多的内容。
##### 常量
```go
const (
	RuneError = '\uFFFD'     // 这是一个特殊的字符，用于表示“非法”的unicode
	RuneSelf  = 0x80         // 十六进制的80，等于十进制的127，0~127刚好表示128个ASCII码。一个字符的字节码大于127，表示不是ASCII，可能是unicode
	MaxRune   = '\U0010FFFF' // 最大的unicode编码
	UTFMax    = 4            // 一个unicode码最大的字节长度
)
```
##### 函数
utf8.DecodeRune(p []byte)(r rune, size int):从字节切片中解析出第一个完整的unicode码，
返回一个`rune`类型的`r`（第一个unicode），和`r`的真实长度size，从代码中可看出，size的值是0~4。简单看一下返回值就好，这不是本文的重点：
```go
func DecodeRune(p []byte) (r rune, size int) {
	n := len(p)
	if n < 1 {
		return RuneError, 0
	}
	p0 := p[0]
	x := first[p0]
	if x >= as {
		// The following code simulates an additional check for x == xx and
		// handling the ASCII and invalid cases accordingly. This mask-and-or
		// approach prevents an additional branch.
		mask := rune(x) << 31 >> 31 // Create 0x0000 or 0xFFFF.
		return rune(p[0])&^mask | RuneError&mask, 1
	}
	sz := x & 7
	accept := acceptRanges[x>>4]
	if n < int(sz) {
		return RuneError, 1
	}
	b1 := p[1]
	if b1 < accept.lo || accept.hi < b1 {
		return RuneError, 1
	}
	if sz == 2 {
		return rune(p0&mask2)<<6 | rune(b1&maskx), 2
	}
	b2 := p[2]
	if b2 < locb || hicb < b2 {
		return RuneError, 1
	}
	if sz == 3 {
		return rune(p0&mask3)<<12 | rune(b1&maskx)<<6 | rune(b2&maskx), 3
	}
	b3 := p[3]
	if b3 < locb || hicb < b3 {
		return RuneError, 1
	}
	return rune(p0&mask4)<<18 | rune(b1&maskx)<<12 | rune(b2&maskx)<<6 | rune(b3&maskx), 4
}
```

**注意以上函数返回值， rune 是一个 int32， 占4字节，size则是返回的unicode占的真实长度**。这里统一用四个字节来表示一个unicode。



### index

这里有一系列的index的函数:
* func Contains(b, subslice []byte) bool
* func Count(s, sep []byte) int
* func Index(s, sep []byte) int
* func IndexByte(s []byte, c byte) int
* func IndexRune(s []byte, r rune) int
* func IndexAny(s []byte, chars string) int
* func IndexFunc(s []byte, f func(r rune) bool) int
* func LastIndex(s, sep []byte) int
* func LastIndexAny(s []byte, chars string) int
* func LastIndexFunc(s []byte, f func(r rune) bool) int
* ...

但是基本上是在调用同一个函数： `func Index(s, sep []byte) int `

```go
func Index(s, sep []byte) int {
	n := len(sep)
	switch {
	case n == 0:
		return 0
	case n == 1:
		return IndexByte(s, sep[0])
	case n == len(s):
		if Equal(sep, s) {
			return 0
		}
		return -1
	case n > len(s):
		return -1
	case n <= shortStringLen:
		// Use brute force when s and sep both are small
		if len(s) <= 64 {
			return indexShortStr(s, sep)
		}
		c := sep[0]
		i := 0
		t := s[:len(s)-n+1]
		fails := 0
		for i < len(t) {
			if t[i] != c {
				// IndexByte skips 16/32 bytes per iteration,
				// so it's faster than indexShortStr.
				o := IndexByte(t[i:], c)
				if o < 0 {
					return -1
				}
				i += o
			}
			if Equal(s[i:i+n], sep) {
				return i
			}
			fails++
			i++
			// Switch to indexShortStr when IndexByte produces too many false positives.
			// Too many means more that 1 error per 8 characters.
			// Allow some errors in the beginning.
			if fails > (i+16)/8 {
				r := indexShortStr(s[i:], sep)
				if r >= 0 {
					return r + i
				}
				return -1
			}
		}
		return -1
	}
	return indexRabinKarp(s, sep)
}
```
可以看到，是几个 case，主要是排除掉一些简单的情况。从`s`中找到第一次出现`sep`的位置，当sep长度为1，sep就是一个字节，
直接调用`IndexByte（s, sep[0]）`,就是把`s`里的字节挨个取出和sep[0]对比。
`case n <= shortStringLen:`表示最普遍的情况，这里使用了相关匹配算法，就不深入谈论了。


再简单看下`func IndexAny(s []byte, chars string) int`函数，返回`s`中第一次出现`chars`中任一字符的位置。
注意chars是string类型，可能包含ASCII和unicode，所以要先对`chars`做处理：
```go
func IndexAny(s []byte, chars string) int {
	if chars == "" {
		// Avoid scanning all of s.
		return -1
	}
  if len(s) > 8 {// 为什么这里判断len(s) > 8,我也不知道，如果你知道麻烦告诉我哦
      // 这里 `makeASCIISet(chars)` 尝试将chars解析成ASCII码并生成集合，如果chars全都是ASCII码，isAscii 为 TRUE，反之为false
		if as, isASCII := makeASCIISet(chars); isASCII {
			for i, c := range s {
				if as.contains(c) {//找到第一个元素，返回下标
					return i
				}
			}
			return -1
		}
	}
  
  // 当len(s)<=8 或者 chars中包含unicode码会执行到这里。以下这段请仔细看，这样的套路会在bytes包里广泛使用。
	var width int  //记录每个字符长度
	for i := 0; i < len(s); i += width {
		r := rune(s[i])
		if r < utf8.RuneSelf {// 表示r是一个ASCII码，则长度为一字节
			width = 1
		} else {
			r, width = utf8.DecodeRune(s[i:]) // 从第 i 个开始到第 i+width 个字节，可能解析成一个unicode
		}
		for _, ch := range chars {//依次取出chars中每一个rune，和r比较
			if r == ch {
				return i
			}
		}
	}
	return -1
}
```
基本上`index`类的函数理解以上两个就够了，其余的是在他们的基础上有小的调节。



### splite

- [func Split(s, sep [\]byte) [][]byte](https://wizardforcel.gitbooks.io/golang-stdlib-ref/content/7.html#Split)
- [func SplitN(s, sep [\]byte, n int) [][]byte](https://wizardforcel.gitbooks.io/golang-stdlib-ref/content/7.html#SplitN)
- [func SplitAfter(s, sep [\]byte) [][]byte](https://wizardforcel.gitbooks.io/golang-stdlib-ref/content/7.html#SplitAfter)
- [func SplitAfterN(s, sep [\]byte, n int) [][]byte](https://wizardforcel.gitbooks.io/golang-stdlib-ref/content/7.html#SplitAfterN)
- ...

Splite 函数根据sep将字节切片切分成很多小切片，然后组成新的切片的切片。

和index函数一样，splite基本上是在调用`func genSplit(s, sep []byte, sepSave, n int) [][]byte`

```go
func genSplit(s, sep []byte, sepSave, n int) [][]byte {
  //s是源切片，sep是分割切片，sepSave表示分割后要保留sep中的多少位，n表示取前n-1个分段，超出n个的部分不分割，直接返回
	if n == 0 {
		return nil
	}
	if len(sep) == 0 {
		return explode(s, n) //长度为0，则按照“空”分割，也就是每个字符都分割，explode函数的功能是将切片分割成字符切片
	}
	if n < 0 {
		n = Count(s, sep) + 1 //重新计算n，表示有多少sep段就分多少次
	}

	a := make([][]byte, n)
	n--
	i := 0
	for i < n {
		m := Index(s, sep)
		if m < 0 {
			break
		}
		a[i] = s[: m+sepSave : m+sepSave]  //m + sepSave
		s = s[m+len(sep):]
		i++
	}
	a[i] = s
	return a[:i+1]
}

```

 

fields 的功能也是分段，先来看一下`FieldsFunc`,该函数通过f()得返回值来判断是否应该在某个字符处分段

```
func FieldsFunc(s []byte, f func(rune) bool) [][]byte {
	// A span is used to record a slice of s of the form s[start:end].
	// The start index is inclusive and the end index is exclusive.
	type span struct {		//用于记录每一段的起始和终点下标
		start int
		end   int
	}
	spans := make([]span, 0, 32)

	// Find the field start and end indices.
	wasField := false			// wasField 表示从fromIndex开始到当前的字符是否应该出现在结果中
	fromIndex := 0
	for i := 0; i < len(s); {
		size := 1
		r := rune(s[i])
		if r >= utf8.RuneSelf {
			r, size = utf8.DecodeRune(s[i:])
		}
		if f(r) {		// r 应该分段
			if wasField {		// 从fromIndex到当前(i)应该出现在结果中
				spans = append(spans, span{start: fromIndex, end: i})
				wasField = false
			}
		} else {
			if !wasField {		// 从fromIndex到当前(i)不应该出现在结果中
				fromIndex = i
				wasField = true
			}
		}
		i += size
	}

	// Last field might end at EOF.
	if wasField {
		spans = append(spans, span{fromIndex, len(s)})
	}

	// Create subslices from recorded field indices.
	a := make([][]byte, len(spans))
	for i, span := range spans {
		a[i] = s[span.start:span.end:span.end]
	}

	return a
}
```

还有一个`Fields`函数，其实就是调用`FieldsFunc`函数，只不过传入的`f`参数是一个判断字符是否是空白符的函数，就不展示代码了



### Map函数

`func Map(mapping func(r rune) rune, s []byte) []byte `是对字节切片中每个字符进行遍历，并且根据传入的`mapping`函数对每个字符进行修改然后存到新的切片中，最后返回新的切片。

`ToUpper`,`ToLower`,`ToTitle`…等函数都调用了Map函数

```go
func Map(mapping func(r rune) rune, s []byte) []byte {
   // In the worst case, the slice can grow when mapped, making
   // things unpleasant. But it's so rare we barge in assuming it's
   // fine. It could also shrink but that falls out naturally.
   maxbytes := len(s) // length of b
   nbytes := 0        // number of bytes encoded in b
   b := make([]byte, maxbytes)
   for i := 0; i < len(s); {
      wid := 1
      r := rune(s[i])
      if r >= utf8.RuneSelf {
         r, wid = utf8.DecodeRune(s[i:])
      }
      r = mapping(r) 		//r 即修改后的字符
      if r >= 0 {
         rl := utf8.RuneLen(r)
         if rl < 0 {
            rl = len(string(utf8.RuneError))
         }
         if nbytes+rl > maxbytes {
            // Grow the buffer.
            maxbytes = maxbytes*2 + utf8.UTFMax
            nb := make([]byte, maxbytes)
            copy(nb, b[0:nbytes])
            b = nb
         }
         nbytes += utf8.EncodeRune(b[nbytes:maxbytes], r)
      }
      i += wid
   }
   return b[0:nbytes]
}
```



### （完）

