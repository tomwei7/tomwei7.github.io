+++
title = "Golang切片使用与内部实现(翻译)"
date = "2016-12-27"
categories = ["golang", "translate"]
tags = ["golang"]
description = "Golang的切片类型提供了一种简单高效的操作序列化类型数据的方式。切片和其他语言中的数组类似，不过含有一些特殊的属性"
image = "golang-arrays-and-slices.png"
+++

Golang的切片类型提供了一种简单高效的操作序列化类型数据的方式。切片和其他语言中的数组类似，不过含有一些特殊的属性。这篇文章我们就来看一下什么是切片和如何使用切片
<!--more-->
### 数组

切片类型是建立在数组类型上的抽象，所以在我们了解切片之前我们必须先了解数组。

数组类型定义了一种特定长度和元素的类型。例如，`[4]int `代表了有四个`int`元素的数组。数组的大小是固定的；它的大小是它类型的一部分（`[4]int`和`[5]int`是不同的，不兼容的类型）。数组可以直接定位，表达式`s[n]`可以直接访问第n个元素，从零开始。

```go
var a [4]int
a[0] = 1
i := a[0]
// i == 1
```

数组不需要显式的初始化；一个零值的数组可以被使用，数组中每个元素是他们自己本身的零值([golang零值参考这里](https://golang.org/ref/spec#The_zero_value))

```go
// a[2] == 0, the zero value of the int type
```

在内存中代表`[4]int`就是4个整形的值顺序排列

![](go-slices-usage-and-internals_slice-array.png)

Golang 中数组是个值。数组的变量代表着整个数组；而不是指向数组第一个元素的指针（这个和C语言中的情况不同）。这代表这当你分配或者传递数组变量时将会拷贝整个数组的内容。（为了避免复制你可以传递一个指向数组的指针，不过这时这就是数组的指针而不是数组了。）你可以把数组想成一种结构体但具有索引而不是命名字段的固定大小的复合值。

一个数组可以这样指定：

```go
b := [2]string{"Penn", "Teller"}
```

或者，你可以让编译器为你计算数组元素：

```go
b := [...]string{"Penn", "Teller"}
```

在这两种情况下，`b`的类型都是`[2]string`

### 切片

数组有它的作用，但是它不够灵活，所以你经常在golang 的代码中见到它。切片，到处都有。切片建立在数组的基础上且更加的强大灵活。

`[]T`是切片类型的定义，其中`T`是切片中元素的类型，跟数组不同，切片类型没有长度。

声明一个切片和声明一个数组有点像，除了不需要指定元素个数

```go
letters := []string{"a", "b", "c", "d"}
```

一个切片可以由内置的`make`函数创建

```go
func make([]T, len, cap) []T
```

其中`T`代表要创建的切片其中元素的类型，`make`函数接受类型、长度和容量选项。调用时，`make`分配一个数组和返回一个指向该数组的切片

```go
var s []byte
s = make([]byte, 5, 5)
// s == []byte{0, 0, 0, 0, 0}
```

当容量参数省略时，默认值等于指定的长度。这个有个更加简洁版本的代码：

```go
s := make([]byte, 5)
```

可以使用内置的`len`和`cap`获取切片的长度和容量

```go
len(s) == 5
cap(s) == 5
```

接下来，我们来看一下切片长度与容量直接的关系

零值的切片是`nil`。`nil`的切片`len`和`cap`函数都会返回0。

还可以通过切片现有的切片或者数组来生成切片。切片通过指定具有两个用冒号分隔的索引的半开范围来完成。比如，表达式`b[1:4]`创建了一个切片包含了`b`中1到3的元素（得带的切片的索引是0到2）。

```go
b := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
// b[1:4] == []byte{'o', 'l', 'a'}, sharing the same storage as b
```

切片表达式的开始和结束索引是可选的;它们分别默认为零和切片的长度：

```go
// b[:2] == []byte{'g', 'o'}
// b[2:] == []byte{'l', 'a', 'n', 'g'}
// b[:] == b
```

通过数组创建切片的语法也一样：

```go
x := [3]string{"Лайка", "Белка", "Стрелка"}
s := x[:] // a slice referencing the storage of x
```

### 切片内部

切片是数组段的描述符。它由指向数组的指针，段的长度及其容量（段的最大长度）组成。

![](go-slices-usage-and-internals_slice-struct.png)

我们先前由`make([]byte,5)`创建的变量`s`的结构如下：

![](go-slices-usage-and-internals_slice-1.png)

长度是切片引用的元素的数量。容量是底层数组中元素的数量（从slice指针引用的元素开始）。将通过下面几个小例子，将帮助你理解长度和容量之间的区别。

在我们切片时，观察切片数据结构中的变化及其与基础数组的关系：

```go
s = s[2:4]
```

!()[https://blog.golang.org/go-slices-usage-and-internals_slice-2.png]

切片操作不会拷贝已有的切片数据，将创建一个新的切片指向原来的数组。这使得切片操作与操作数组索引一样高效。因此，修改重新切片的元素（而不是切片本身）将修改原始切片的元素：

```go
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:]
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

之前我们把`s`切片成长度比容量少的切片，我们可以通过重新切片将它长度增长和容量一样大(我之前还真不知道能这么玩😂)：

```go
s = s[:cap(s)]
```

切片不能超越其容量。尝试这样做会导致运行时panic，就像在切片或数组的边界外建立索引一样。类似地，切片不能在零以下重新切片以访问数组中的早期元素。

### 切片增长（拷贝和append函数）

为了增加切片的容量，必须创建一个新的较大切片，并将原始切片的内容复制到其中。这种技术就是其他语言的动态数组的背后实现。下一个例子通过制作一个新的切片`t`，将`s`的内容复制到`t`，然后将切片值`t`分配给`s`来使`s`的容量加倍：

```go
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
        t[i] = s[i]
}
s = t
```

通过内置的`copy`函数可以更加容易的实现这种循环操作。顾名思义，复制将数据从源切片复制到目标切片。它返回复制的元素数。

复制功能支持在不同长度的切片之间进行复制（它只会复制到较少数量的元素）。此外，副本可以处理共享相同底层数组的源和目标slice，正确处理重叠slice。

使用copy，我们可以简化上面的代码片段：

```go
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```
常见的操作是将数据附加到切片的末尾。此函数将字节元素附加到字节片，如果必要，增长切片，并返回新的切片：

```go
func AppendByte(slice []byte, data ...byte) []byte {
    m := len(slice)
    n := m + len(data)
    if n > cap(slice) { // if necessary, reallocate
        // allocate double what's needed, for future growth.
        newSlice := make([]byte, (n+1)*2)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:n]
    copy(slice[m:n], data)
    return slice
}
```

可以使用AppendByte这样：

```go
p := []byte{2, 3, 5}
p = AppendByte(p, 7, 11, 13)
// p == []byte{2, 3, 5, 7, 11, 13}
```

像AppendByte这样的函数很有用，因为它们提供对切片生长方式的完全控制。根据程序的特性，可能需要分配更小或更大的块，或者对重新分配的大小设置上限。

但是大多数程序不需要完全控制，所以Go提供了一个内置的`append`函数，适合大多数用途; 函数声明：

```go
func append(s []T, x ...T) []T
```

`append`函数将元素x附加到切片的末尾，如果需要更大的容量，则增长切片。

```go
a := make([]int, 1)
// a == []int{0}
a = append(a, 1, 2, 3)
// a == []int{0, 1, 2, 3}
```

要将一个切片附加到另一个切片，请使用...将第二个参数展开为参数列表。

```go
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

由于切片的零值（nil）的行为类似于零长度切片，因此你可以声明切片变量，然后在循环中附加切片变量：

```go
// Filter returns a new slice holding only
// the elements of s that satisfy f()
func Filter(s []int, fn func(int) bool) []int {
    var p []int // == nil
    for _, v := range s {
        if fn(v) {
            p = append(p, v)
        }
    }
    return p
}
```

### 一个小问题

如前所述，重新切片切片不会生成底层数组的副本。完整数组将保存在内存中，直到它不再被引用。有时可能导致只需要数组中一点点内容，但是整个数组都被保存在内存中。
例如，此FindDigits函数将文件加载到内存中，并搜索第一组连续数字，将其作为新切片返回。

```go
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

此代码的行为与期望一样，但返回的`[]byte`指向包含整个文件的数组。因为切片引用原始数组，只要切片被保留,垃圾收集器就不能释放数组；文件的整个内容保存在内存中但有用的只有一点。

要解决这个问题，可以将需要的数据复制到一个新的切片，然后返回它：

```go
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

### 参考
原文[Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
