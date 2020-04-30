---
title: golang反射三法则
date: 2018-07-22 19:26:20
tags: golang
---
近期在为bingo框架添砖加瓦，用到了golang的反射。这篇文章解决了我不少疑难杂症，特分享给大家。

<!--more-->
### 引言
在计算机科学里，反射是程序，特别是通过类型，查看并操纵其自身结构的能力。 它是元编程的一种形式，同时也是最容易让人误解的部分。
在本文中，我们试图解释反射在 Go 中是如何工作，希望能澄清对它的误解。每一种语言的发射模型都不同 （甚至许多语言根本不支持反射），不过既然这篇文章是关于 Go 的，因此在接下来的内容中， “反射”一词应理解成“Go 中的反射”。
### 类型与接口
由于反射建立在类型系统之上，就让我先来复习一下 Go 语言里的类型吧。
Go是静态类型的语言。每个变量都有一种静态类型。换言之，他们的类型在 编译期就确定并且固定下来了。 比如 int 、 float32 、 \*MyType 或 []byte 等等。如果我们定义了
````go
type MyInt int
var i int
var j MyInt
````
那么i 的类型为 int ，而 j 的类型为 MyInt 。尽管变量 i 和 j 拥有相同的基本类型， 但它们的静态类型仍然不同，因此在它们未经转换前是不能相互赋值的。
在类型中，有一种重要的类别就是接口类型，它表示一个确定的方法集。只要某个具体值 （非接口）实现了某个接口中的方法，该接口类型的变量就能存储它。一对众所周知的例子就是 io.Reader 和 io.Writer ，即 io 包中的 Reader 和 Writer 类型：
```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}
// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```
任何实现了 Read （或 Write ）方法及其签名的类型， 同时也就实现了 io.Reader （或 io.Writer ）接口。 就此而言，若某个值的类型拥有 Read 方法， io.Reader 类型的变量就能保存它：
```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
```
有件事一定要明确，即无论 r 保存了什么具体的值， r 的类型总是 io.Reader ：Go是静态类型的，而 r 的静态类型为 io.Reader 。
一个非常重要的接口类型是空接口：
```go
interface{}
```
它表示空方法集。由于任何值都有零或多个方法，因此任何值都满足它。
有人说Go的接口是动态类型的，不过这是种误解。它们确实是静态类型的： 接口类型的变量总有着相同的静态类型，就算存储在其中的值的类型在运行时可能会改变， 它也总是满足该接口。
对于所有的这些，我们都必须严谨对待，因为反射和接口密切相关。
### 接口的表示
Russ Cox 写过一篇题为[Go中接口值的表示](http://research.swtch.com/2009/12/go-data-structures-interfaces.html)的文章，我们就不在此赘述了。不过简单概括一下还是很有必要的。
接口类型的变量存储了一对内容：赋予该变量的具体值，以及该值的类型描述符。 更准确地说，其值是实现了该接口的具体数据条目，而其类型则描述了该条目的完整类型。 例如，在执行
```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```
之后，如果按形式来描述，那么 r 就包含了 (值, 类型) 对，即 ( tty, \*os.File )。注意，类型 \*os.File 还实现了除 Read 以外的其它方法：尽管该接口值只提供了访问 Read 方法的能力，但其内部却携带了有关该值的所有类型信息。 这就是我们可以这样做的原因：
```go
var w io.Writer
w = r.(io.Writer)
```
在此赋值语句中的表达式是一个类型断言：它断言 r 内的条目同时也实现了 io.Writer ，因此我们可以将它赋予 w 。 赋值后， w 将会包含一对 ( tty , \*os.File )。 这与保存在 r 中的一致。接口的静态类型决定了哪些方法可通过接口变量调用， 即便其内部具体的值可能有更大的方法集。
接着，我们可以这样做：
```go
var empty interface{}
empty = w
```
而我们的空接口值 e 也将再次包含同样的一对 ( tty, \*os.File )。 这很方便：空接口可保存任何值，并包含关于该值的所有信息。
（在这里我们无需类型断言，因为 w 肯定是满足空接口的。在本例中， 我们将一个值从 Reader 变成了 Writer ，由于 Writer 的方法并非 Reader 的子集，因此我们必须显式地使用类型断言。）
一个很重要的细节，就是接口内部的对总是 (值, 具体类型) 的形式，而不会是 (值, 接口类型) 的形式。接口不能保存接口值。
现在我们准备好聊聊反射了。
### 反射法则一：反射是从接口值到反射对象
从基本层面上看，反射只是一种检查存储在接口变量中的“类型-值对”的机制。 首先，我们需要了解reflect包中的两种类型：Type和Value。这两种类型可用来访问接口变量的内容。 还有两个简单的函数，叫做reflect.TypeOf 和 reflect.ValueOf ， 它们用于接口值中分别获取 reflect.Type 和 reflect.Value 。 同样，从 reflect.Value 也能很容易地获取 reflect.Type ， 不过让我们先保持 Value 和 Type 概念的独立性吧。
我们先从 TypeOf 开始：
```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```
此程序会打印出：
```go
type: float64
```
你可能会问接口在哪，因为该程序看起来只是向 reflect.TypeOf 传递了一个 float64 类型的变量 x ，而不是一个接口值。但它就在那， reflect.TypeOf 的签名包含了一个空接口：
```go
// TypeOf 返回 interface{} 中的值的反射类型 Type。
func TypeOf(i interface{}) Type
```
当我们调用 reflect.TypeOf(x) 时， x 首先会被存储在一个空接口中， 然后它会作为实参被传入； reflect.TypeOf 通过解包该空接口来还原其类型信息。
当然， reflect.ValueOf 函数也会还原它的值（从这里开始， 我们会略过那些概念示例，而只关注于可执行的代码）：
```go
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x))
```
会打印出
```go
value: 3.4
```
reflect.Type 和 reflect.Value 都有许多方法来让我们检测并操作它们。 一个重要的例子就是 Value 拥有一个 Type 方法，它会返回 reflect.Value 的 Type 。另外就是 Type 和 Value 都有一个 Kind 方法，它会返回一个常量来表明条目的类型： Uint 、 Float64 或 Slice 等等。同样， Value 拥有像 Int 和 Float 这样的方法来让我们获取存储在内部的值 （作为 int64 和 float64 ）:
```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```
会打印出
```go
type: float64
kind is float64: true
value: 3.4
```
同样还有 SetInt 和 SetFloat 这样的方法，不过在使用它们之前， 我们需要理解其可设置性，该主题会在后面的第三条反射法则中讨论。
反射库有几点特性值得一提。首先，为了让 API 保持简单， Value 的“获取者”和“设置者”方法会在能够保存其值的最大类型上进行操作：例如 int64 就能用于所有的带符号整数。也就是说， Value 的 Int 方法会返回 int64 类型的值，而 SetInt 会接受 int64 类型的值；因此该值可能需要转换为它所涉及到的实际类型：
```go
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())                                       // v.Uint returns a uint64.
```
第二个特性就是反射对象的 Kind 描述了其基本类型，而给静态类型。 若反射对象包含了用户定义的整数类型的值，比如
```go
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
```
那么 v 的 Kind 仍为 reflect.Int ，尽管 x 的静态类型为 MyInt 而非 int 。换句话说， Kind 无法区分 int 和 MyInt ，而 Type 则可以。
### 反射法则二：从反射对象可反射出接口值
如同物理中的反射现象那样，Go中的反射也会产生它自己的镜像。
给定一个 reflect.Value ，我们可以使用 Interface 方法还原其接口值；在效果上，该方法会将类型与值的信息打包成接口表示，并返回其结果：
```go
// Interface 将 v 的值返回成 interface{}。
func (v Value) Interface() interface{}
```
因此，我们可以通过
```go
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```
打印出反射对象 v 所表示的 float64 值。
不过我们可以做得更好。 fmt.Println 与 fmt.Printf 等都会将实参作为空接口值传递，它们会被包 fmt 进行内部解包， 就像我们刚做的那样。因此，正确地打印出 reflect.Value 内容的方法就是 将 Interface 方法的结果传至格式化打印功能：
```go
fmt.Println(v.Interface())
```
（为什么不是 fmt.Println(v) ？因为 v 是个 reflect.Value ，而我们想要的是它保存的具体值。）由于值的类型是 float64 ，如果需要的话，我们甚至可以使用浮点数格式化：
```go
fmt.Printf("value is %7.1e\n", v.Interface())
```
然后就会得到
```go
3.4e+00
```
再次强调，这里无需将 v.Interface() 的结果类型断言为 float64 ， 因为空接口值中拥有具体值的类型信息，而 Printf 则会将它还原。
简单来说， Interface 方法就是 ValueOf 函数的镜像， 不过其结果总是静态类型 interface{} 。
重申一遍：从接口值可反射出反射对象，反之亦可。
### 反射法则三：可设置性
第三条法则是最微妙而令人困惑的，但如果我们从第一条法则开始，还是很容易理解的。
这些代码虽然不能工作，但很值得学习（自己也在这里踩过坑）。
```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```
如果你运行这段代码，它就会报出神秘的恐慌信息：
```go
panic: reflect.Value.SetFloat using unaddressable value
```
其问题的根源不在于值 7.1 能不能寻址，而在于 v 不可设置。 可设置性是反射值 Value 的一种属性，而且并不是所有的反射值都拥有它。
Value 的 CanSet 方法会报告 Value 的可设置性。 在我们的例子中，
```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```
会打印出
```go
settability of v: false
```
对不可设置的 Value 调用 Set 方法会产生错误，但什么是可设置性呢？
可设置性有点像可寻址性，不过它更加严格。它是反射对象能否修改其创建之初的真实值的一种属性。 可设置性决定了反射对象是否保存原始条目。当我们执行
```go
var x float64 = 3.4
v := reflect.ValueOf(x)
```
后，就将 x 的一份副本传入了 reflect.ValueOf ， 因此该接口值也就作为传递给 reflect.ValueOf 的实参创建了一份 x 的副本，而非 x 本身。因此，假如语句
```go
v.SetFloat(7.1)
```
能够成功执行，它也无法更新 x ，即便 v 看起来创建自 x 。就算它能够更新存储在该反射值中的 x 的副本， x 本身也不会受影响。这是令人困惑且毫无用处的，因此它是非法的。 而可设置性就是用于避免此类问题的属性。
这看起来很奇怪，事实却并非如此。它其实就是藏在奇特外表下的一种常见情况。 考虑将 x 传递给一个函数f(x)。
我们并不期望 f 能修改 x ，因为我们传入的是值 x 的副本，而非 x 本身。如果我们想让 f 直接修改 x ，就必须向该函数传入 x 的地址（即指向 x 的指针）：f(&x)。
这即熟悉又简单，反射也是以相同的方式工作的。如果我们想要通过反射修改 x ，就必须向反射库提供要修改的值的指针。
让我们试试吧。首先像平时那样初始化 x ，接着创建指向它的反射值，叫做 p 。
```go
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```
目前会输出
```go
type of p: *float64
settability of p: false
```
反射对象 p 并不是可设置的，不过我们也不想设置 p ， 而（实际上）是 \*p 。为获得 p 指向的内容，我们调用 Value 的 Elem 方法，它会间接通过指针，并将结构保存到叫做 v 的反射值 Value 中：
```go
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```
现在 v 是可设置的反射对象，如输出所示：
```go
settability of v: true
```
由于它代表 x ，因此最终我们可使用 v.SetFloat 来修改 x 的值：
```go
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```
并得到期望的输出：
```go
7.1
7.1
```
反射可能很难理解，但语言做了它应该做的，尽管反射类型 Types 和值 Values 隐藏了发生的事情。只要记得反射值需要某些东西的地址来修改它所代表的东西。
### 结构体
在我们前面的例子中， v 本身并不是指针，它只是从一个指针中获取的。 在使用反射修改结构体的字段时，这种情况经常出现。即，当我们有结构体的地址时， 就能修改它的字段。
下面这个简单的例子分析了结构体类型的值 t 。我们从它的地址创建了反射对象， 因为待会儿要修改它。接着我们将 typeOfT 设置为它的类型， 然后以直白的方法调用遍历其字段（详见 [reflect](https://go-zh.org/pkg/reflect/) 包）。 注意，我们从该结构体类型中提取了其字段名，但字段本身是一般的 reflect.Value 对象。
```go
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```
此程序的输出为：
```go
0: A int = 23
1: B string = skidoo
```
这里还有一个关于可设置性的要点： T 的字段名必须大写（已导出）， 因为只有已导出的字段才是可设置的。
由于 s 包含了可设置的反射对象，因此我们可以修改该结构体的字段：
```go
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```
其结果为：
```go
t is now {77 Sunset Strip}
```
若我们将此程序修改为 s 创建自 t 而非 &t ，那么调用 SetInt 和 SetString 就会失败，因为 t 的字段不可设置。
### 总结
再次提示，反射法则如下：
- 从接口值可反射出反射对象。
- 从反射对象可反射出接口值。
- 要修改反射对象，其值必须可设置。

一旦你理解了Go中的这些反射法则，它就会变得更容易使用了，尽管它还是很微妙。 这是个强大的工具，因此除非有必要，否则应当避免或小心使用。

