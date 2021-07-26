# 值部

此篇文章后续的若干文章将介绍Go中更多的类型。为了更容易和更深刻地理解那些类型，最好先阅读一下本文。

### Go类型分为两大类别（category）

Go可以被看作是一门C语言血统的语言，这可以通过此前的[指针](https://gfw.go101.org/article/pointer.html)和[结构体](https://gfw.go101.org/article/struct.html)两篇文章得以验证。 Go中的指针和结构体类型的内存结构和C语言很类似。

另一方面，Go也可以被看作是C语言的一个扩展框架。这可以从C中的值的内存结构都是很透明的，但Go中一些种类的类型的值的内存结构却不是很透明这一事实体现出来。 在C中，每个值在内存中只占据一个[内存块](https://gfw.go101.org/article/memory-block.html)（一段连续内存）。但是，一些Go类型的值可能占据多个内存块。

以后，我们称一个Go值分布在不同内存块上的部分为此值的各个值部（value part）。 一个分布在多个内存块上的值含有一个直接值部和若干被此直接值部[引用着](https://gfw.go101.org/article/pointer.html#references)的间接值部。

上面的段落描述了两个类别的Go类型。下表将列出这两个类别（category）中的类型（type）种类（kind）：

|           每个值在内存中只分布在一个内存块上的类型           |           每个值在内存中会分布在多个内存块上的类型           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![单值部](https://gfw.go101.org/article/res/value-parts-single.png) | ![多值部](https://gfw.go101.org/article/res/value-parts-multiple.png) |
| 布尔类型 各种数值类型 指针类型 非类型安全指针类型 结构体类型 数组类型 |   切片类型 映射类型 通道类型 函数类型 接口类型 字符串类型    |

表中列出的很多类型将在后续文章中逐一详细讲解。本文的目的就是为了给后续的讲解做一个铺垫。

注意：

- **接口类型**和**字符串类型**值是否包含间接部分取决于**具体编译器**实现。 如果不使用今后将介绍的非类型安全途径，我们无法从这两类类型的值的外在表现来判定它们的值是否含有间接部分。 在《Go语言101》中，我们认为这两类类型的值是**可能包含间接值部**的。
- 同样地，函数类型的值是否包含间接部分几乎也是不可能验证的。 在《Go语言101》中，我们认为**函数值是可能包含间接值部**的。

通过封装了很多具体的实现细节，第二个类别中的类型给Go编程带来了很大的便利。 **不同的编译器实现会采用不同的内部结构来实现这些类型，但是这些类型的值的外在表现必须满足Go白皮书中的要求**。 此分类中的类型对于编程来说并非是很基础的类型。 我们可以使用第一个分类中的类型来实现此分类中的类型。 但是，通过将一些常用或者很独特的功能封装到此第二个分类中的类型里，使用Go编程的效率将得到大大提升，体验将得到大大增强。

另一方面，这些封装同时也隐藏了这些类型的值的内部结构，使得Go程序员不能对这些类型有一个更全局更深刻的认识。有时候这会对更好地理解Go带来了一些障碍。

为了帮助Go程序员更好的理解第二个分类中的类型和它们的值，本文余下的内容将对这些类型的内在实现做一个简单介绍。 这些实现的细节将不会在本文中谈及。本文的介绍主要基于（但并不完全符合）官方标准编译器的实现。

### Go中的两种指针类型

在继续下面的内容之前，我们先了解一下Go中的两种指针类型并明确一下“引用”这个词的含义。

我们已经在[上上篇文章](https://gfw.go101.org/article/pointer.html)中了解了Go中的指针。 那篇文章中所介绍的指针属于**类型安全的指针**。事实上，Go还支持另一种称为**[非类型安全的指针类型](https://gfw.go101.org/article/unsafe.html)**。 非类型安全的指针类型提供在`unsafe`标准库包中。 非类型安全指针类型通常使用`unsafe.Pointer`来表示。 `unsafe.Pointer`类似于C语言中的`void*`。

在《Go语言101》中的大多数文章中，如果没有特别说明，当一个指针类型被谈及，它表示一个类型安全指针。 但是在本文的余下内容中，当一个指针被谈及，它可能表示一个类型安全指针，也可能表示一个非类型安全指针。

一个指针值存储着另一个值的地址，除非此指针值是一个nil空指针。 我们可以说此指针[引用着](https://gfw.go101.org/article/pointer.html#references)另外一个值，或者说另外一个值正被此指针所引用。 一个值可能被间接引用，比如

- 如果一个结构体值`a`含有一个指针字段`b`并且这个指针字段`b`引用着另外一个值`c`，那么我们可以说结构体值`a`也引用着值`c`。
- 如果一个值`x`（直接或者间接地）引用着另一个值`y`，并且值`y`（直接或者间接地）引用着第三个值`z`，则我们可以说值`x`间接地引用着值`z`。

以后，我们将一个含有（直接或者间接）指针字段的结构体类型称为一个**指针包裹类型**，将一个含有（直接或者间接）指针的类型称为**指针持有者类型**。 指针类型和指针包裹类型都属于指针持有者类型。元素类型为指针持有者类型的数组类型也是指针持有者类型（数组将在下一篇文章中介绍）。



### 第二个分类中的类型的（可能的）内部实现结构定义

为了更好地理解第二个分类中的类型的值的运行时刻行为，我们可以认为这些类型在内部是使用第一个分类中的类型来定义的（如下所示）。 如果你以前并没有很多使用过Go中各种类型的经验，目前你不必深刻地理解这些定义。 对这些定义拥有一个粗糙的印象足够对理解后续文章中将要讲解的类型有所帮助。 你可以在今后有了更多的Go编程经验之后再重读一下本文。

#### 映射、通道和函数类型的内部定义

映射、通道和函数类型的内部定义很相似：

```go
// 映射类型
type _map *hashtableImpl // 目前，官方标准编译器是使用
                         // 哈希表来实现映射的。

// 通道类型
type _channel *channelImpl

// 函数类型
type _function *functionImpl
```

从这些定义，我们可以看出来，这三个种类的类型的内部结构其实是一个**指针类型**。 或者说，这些类型的值的直接部分在内部是一个指针。 这些类型的每个值的直接部分引用着它的具体实现的底层间接部分。

#### 切片类型的内部定义

切片类型的内部定义：

```go
type _slice struct {
	elements unsafe.Pointer // 引用着底层的元素
	len      int            // 当前的元素个数
	cap      int            // 切片的容量
}
```

从这个定义可以看出来，一个切片类型在内部可以看作是一个指针包裹类型。 每个非零切片值包含着一个底层间接部分用来存储此切片的元素。 一个切片值的底层元素序列（间接部分）被此切片值的`elements`字段所引用。

#### 字符串类型的内部结构

```go
type _string struct {
	elements *byte // 引用着底层的byte元素
	len      int   // 字符串的长度
}
```

从此定义可以看出，每个字符串类型在内部也可以看作是一个指针包裹类型。 每个非零字符串值含有一个指针字段 `elements`。 这个指针字段引用着此字符串值的底层字节元素序列。



#### 接口类型的内部定义

我们可以认为接口类型在内部是如下定义的：

```go
type _interface struct {
	dynamicType  *_type         // 引用着接口值的动态类型
	dynamicValue unsafe.Pointer // 引用着接口值的动态值
}
```

从这个定义来看，接口类型也可以看作是一个指针包裹类型。一个接口类型含有两个指针字段。 每个非零接口值的（两个）间接部分分别存储着此接口值的动态类型和动态值。 这两个间接部分被此接口值的直接字段`dynamicType`和`dynamicValue`所引用。

事实上，上面这个内部定义只用于表示空接口类型的值。空接口类型没有指定任何方法。 后面的[接口](https://gfw.go101.org/article/interface.html)一文详细解释了接口类型和值。 非空接口类型的内部定义如下：

```go
type _interface struct {
	dynamicTypeInfo *struct {
		dynamicType *_type       // 引用着接口值的动态类型
		methods     []*_function // 引用着动态类型的对应方法列表
	}
	dynamicValue unsafe.Pointer // 引用着动态值
}
```

一个非空接口类型的值的`dynamicTypeInfo`字段的`methods`字段引用着一个方法列表。 此列表中的每一项为此接口值的动态类型上定义的一个方法，此方法对应着此接口类型所指定的一个的同原型的方法。



### 在赋值中，底层间接值部将不会被复制

现在我们了解了第二个分类中的类型的内部结构是一个指针持有（指针或者指针包裹）类型。 这对于我们理解Go中的值复制行为有很大帮助。

在Go中，每个赋值操作（包括函数调用传参等）都是一个值的**浅复制**过程（假设源值和目标值的类型相同）。 换句话说，在一个赋值操作中，只有源值的直接部分被复制给了目标值。 如果源值含有间接部分，则在此赋值操作完成之后，目标值和源值的直接部分将引用着相同的间接部分。 换句话说，**两个值将共享底层的间接值部**，如下图所示：

![值复制](https://gfw.go101.org/article/res/value-parts-copy.png)

```
注：此处需要重点注意。尤其对于复制后需要对新的变量进行修改的地方，会导致旧变量的底层间接值部也被修改，这种情况需要使用深拷贝解决。
```



事实上，对于字符串值和接口值的赋值，上述描述在理论上并非百分百正确。 [官方FAQ](https://golang.google.cn/doc/faq#pass_by_value)明确说明了在一个接口值的赋值中，接口的底层动态值将被复制到目标值。 但是，因为一个接口值的动态值是只读的，所以在接口值的赋值中，官方标准编译器并没有复制底层的动态值。这可以被视为是一个编译器优化。 对于字符串值的赋值，道理是一样的。所以对于官方标准编译器来说，上一段的描述是100%正确的。

因为一个间接值部可能并不专属于任何一个值，所以在使用`unsafe.Sizeof`函数计算一个值的尺寸的时候，此值的间接部分所占内存空间未被计算在内。



### 关于术语“引用类型”和“引用值”

“引用”这个术语在Go社区中使用得有些混乱。很多Go程序员在Go编程中可能由此产生了一些困惑。 一些文档或者网络文章，包括一些[官方文档](https://golang.google.cn/doc/faq#references)，把“引用”（reference）看作是“值”（value）的一个对立面。 《Go语言101》强烈不推荐这种定义。在这一点上，本人不想争论什么。这里仅仅列出一些肯定**错误地使用**了“引用”这个术语的例子：

- 在Go中，只有切片、映射、通道和函数类型属于***引用类型\***。 （如果我们确实需要***引用类型\***这个术语，那么我们不应把其它指针持有者类型排除在引用类型之外。）
- 一些函数调用的参数是通过引用来传递的。 （对不起，在Go中，所有的函数调用的参数都是通过值复制**直接值部**的方式来传递的。）

我并不是想说***引用类型\***这个术语在Go中是完全没有价值的， 我只是想表达这个术语是完全没有必要的，并且它常常在Go的使用中导致一些困惑。我推荐使用指针持有者类型来代替这个术语。 另外，我个人的观点是最好将***引用\***这个词限定到只表示值之间的关系，把它当作一个动词或者名词来使用，永远不要把它当作一个形容词来使用。 这样将在使用Go的过程中避免很多困惑。

# 数组、切片和映射

在严格意义上，Go中有三种一等公民容器类型：数组、切片和映射。 有时候，我们可以认为字符串类型和通道类型也属于容器类型。 但是，此篇文章只谈及数组、切片和映射类型。

Go中有很多和容器类型相关的细节，本文将逐一列出这些细节。

### 容器类型和容器值概述

每个容器（值）用来表示和存储一个元素（element）序列或集合。一个容器中的所有元素的类型是相同的。此相同的类型称为此容器的类型的元素类型（或简称此容器的元素类型）。

存储在一个容器中的每个元素值都关联着一个键值（key）。每个元素可以通过它的键值而被访问到。 一个映射类型的键值类型必须为一个[可比较类型](https://gfw.go101.org/article/type-system-overview.html#types-not-support-comparison)。 数组和切片类型的键值类型均为内置类型`int`。 一个数组或切片的一个元素对应的键值总是一个非负整数下标，此非负整数表示该元素在该数组或切片所有元素中的顺序位置。此非负整数下标亦常称为一个元素索引（index）。

每个容器值有一个长度属性，用来表明此容器中当前存储了多少个元素。 一个数组或切片中的每个元素所关联的非负整数索引键值的合法取值范围为左闭右开区间`[0, 此数组或切片的长度)`。 一个映射值类型的容器值中的元素关联的键值可以是任何此映射类型的键值类型的任何值。

这三种容器类型的值在使用上有很多的差别。这些差别多源于它们的内存定义的差异。 通过上一篇文章[值部](https://gfw.go101.org/article/value-part.html)，我们得知每个数组值仅由一个直接部分组成，而一个切片或者映射值是由一个直接部分和一个可能的被此直接部分引用着的间接部分组成。

一个数组或者切片的所有元素紧挨着存放在一块连续的内存中。一个数组中的所有元素均存放在此数组值的直接部分，一个切片中的所有元素均存放在此切片值的间接部分。 在官方标准编译器和运行时中，映射是使用哈希表算法来实现的。所以一个映射中的所有元素也均存放在一块连续的内存中，但是映射中的元素并不一定紧挨着存放。 另外一种常用的映射实现算法是二叉树算法。无论使用何种算法，一个映射中的所有元素的键值也存放在此映射值（的间接部分）中。

我们可以通过一个元素的键值来访问此元素。 对于这三种容器，元素访问的时间复杂度均为`*O*(1)`。 但是一般来说，映射元素访问消耗的时长要数倍于数组和切片元素访问消耗的时长。 但是映射相对于数组和切片有两个优点：

- 映射的键值类型可以是任何可比较类型。
- 相对于使用含有大量稀疏索引的数组和切片，使用映射可以节省大量的内存。

从上一篇文章中，我们已经了解到，在任何赋值中，源值的底层间接部分不会被复制。 换句话说，当一个赋值结束后，一个含有间接部分的源值和目标值将共享底层间接部分。 这就是数组和切片/映射值会有很多行为差异（将在下面逐一介绍）的原因。

### 非定义容器类型的字面表示形式

非定义容器类型的字面表示形式如下：

- 数组类型：`[N]T`
- 切片类型：`[]T`
- 映射类型：`map[K]T`

其中，

- `T`可为任意类型。它表示一个容器类型的元素类型。某个特定容器类型的值中只能存储此容器类型的元素类型的值。
- `N`必须为一个非负整数常量。它指定了一个数组类型的长度，或者说它指定了此数组类型的任何一个值中存储了多少个元素。 一个数组类型的长度是此数组类型的一部分。比如`[5]int`和`[8]int`是两个不同的类型。
- `K`必须为一个[可比较类型](https://gfw.go101.org/article/type-system-overview.html#types-not-support-comparison)。它指定了一个映射类型的键值类型。

下面列出了一些非定义容器类型的字面表示：

```go
const Size = 32

type Person struct {
	name string
	age  int
}

// 数组类型
[5]string
[Size]int
[16][]byte  // 元素类型为一个切片类型：[]byte
[100]Person // 元素类型为一个结构体类型：Person

// 切片类型
[]bool
[]int64
[]map[int]bool // 元素类型为一个映射类型：map[int]bool
[]*int         // 元素类型为一个指针类型：*int

// 映射类型
map[string]int
map[int]bool
map[int16][6]string     // 元素类型为一个数组类型：[6]string
map[bool][]string       // 元素类型为一个切片类型：[]string
map[struct{x int}]*int8 // 元素类型为一个指针类型：*int8；
                        // 键值类型为一个结构体类型。
```



所有切片类型的[尺寸](https://gfw.go101.org/article/type-system-overview.html#value-size)都是一致的，所有映射类型的尺寸也都是一致的。 一个数组类型的尺寸等于它的元素类型的尺寸和它的长度的乘积。长度为零的数组的尺寸为零；元素类型尺寸为零的任意长度的数组类型的尺寸也为零。



### 容器字面量的表示形式

和结构体值类似，容器值的文字表示也可以用组合字面量形式（composite literal）来表示。 比如对于一个容器类型`T`，它的值可以用形式`T{...}`来表示（除了切片和映射的零值外）。 下面是一些容器字面量：

```go
// 一个含有4个布尔元素的数组值。
[4]bool{false, true, true, false}

// 一个含有三个字符串值的切片值。
[]string{"break", "continue", "fallthrough"}

// 一个映射值。
map[string]int{"C": 1972, "Python": 1991, "Go": 2009}
```

映射组合字面量中大括号中的每一项称为一个键值对（key-value pair），或者称为一个条目（entry）。

数组和切片组合字面量有一些微小的变种：

```go
// 下面这些切片字面量都是等价的。
[]string{"break", "continue", "fallthrough"}
[]string{0: "break", 1: "continue", 2: "fallthrough"}
[]string{2: "fallthrough", 1: "continue", 0: "break"}
[]string{2: "fallthrough", 0: "break", "continue"}

// 下面这些数组字面量都是等价的。
[4]bool{false, true, true, false}
[4]bool{0: false, 1: true, 2: true, 3: false}
[4]bool{1: true, true}
[4]bool{2: true, 1: true}
[...]bool{false, true, true, false}
[...]bool{3: false, 1: true, true}
```

上例中最后两行中的`...`表示让编译器推断出相应数组值的类型的长度。

从上面的例子中，我们可以看出数组和切片组合字面量中的索引下标（即数组和切片的键值）是可选的。 在一个数组或者切片组合字面量中：

- 如果一个索引下标出现，它的类型不必是数组和切片类型的键值类型`int`，但它必须是一个可以表示为int值的非负常量； 如果它是一个类型确定值，则它的类型必须为一个基本整数类型。
- **在一个数组或切片组合字面量中，如果一个元素的索引下标缺失，则编译器认为它的索引下标为出现在它之前的元素的索引下标加一。**
- 如果出现的第一个元素的索引下标缺失，则它的索引下标被认为是0。

映射组合字面量中元素对应的键值不可缺失，并且它们可以为非常量。

```go
var a uint = 1
var _ = map[uint]int {a : 123} // 没问题
var _ = []int{a: 100}          // error: 下标必须为常量
var _ = [5]int{a: 100}         // error: 下标必须为常量
```



一个容器组合字面量中的[常量键值（包括索引下标）不可重复](https://gfw.go101.org/article/details.html#constant-keys-in-composite-literals)。

### 容器类型零值的字面量表示形式

和结构体类似，一个数组类型`A`的零值可以表示为`A{}`。 比如，数组类型`[100]int`的零值可以表示为`[100]int{}`。 一个数组零值中的所有元素均为对应数组元素类型的零值。

和指针一样，所有切片和映射类型的零值均用预声明的标识符`nil`来表示。

顺便说一句，除了刚提到的三种类型，以后将介绍的函数、通道和接口类型的零值也用预声明的标识符`nil`来表示。

在运行时刻，即使一个数组变量在声明的时候未指定初始值，它的元素所占的内存空间也已经被开辟出来。 但是一个nil切片或者映射值的元素的内存空间尚未被开辟出来。

注意：`[]T{}`表示类型`[]T`的一个空切片值，它和`[]T(nil)`是不等价的。 同样，`map[K]T{}`和`map[K]T(nil)`也是不等价的。



### 容器字面量是不可寻址的但可以被取地址

我们已经了解到[结构体（组合）字面量是不可寻址的但却是可以被取地址的](https://gfw.go101.org/article/struct.html#take-composite-literal-address)。 容器字面量也不例外。

一个例子：

```go
package main

import "fmt"

func main() {
	pm := &map[string]int{"C": 1972, "Go": 2009}
	ps := &[]string{"break", "continue"}
	pa := &[...]bool{false, true, true, false}
	fmt.Printf("%T\n", pm) // *map[string]int
	fmt.Printf("%T\n", ps) // *[]string
	fmt.Printf("%T\n", pa) // *[4]bool
}
```





### 内嵌组合字面量可以被简化

在某些情形下，内嵌在其它组合字面量中的组合字面量可以简化为`{...}`（即类型部分被省略掉了）。 内嵌组合字面量前的取地址操作符`&`有时也可以被省略。

比如，下面的组合字面量

```go
// heads为一个切片值。它的类型的元素类型为*[4]byte。
// 此元素类型为一个基类型为[4]byte的指针类型。
// 此指针基类型为一个元素类型为byte的数组类型。
var heads = []*[4]byte{
	&[4]byte{'P', 'N', 'G', ' '},
	&[4]byte{'G', 'I', 'F', ' '},
	&[4]byte{'J', 'P', 'E', 'G'},
}
```

可以被简化为

```go
var heads = []*[4]byte{
	{'P', 'N', 'G', ' '},
	{'G', 'I', 'F', ' '},
	{'J', 'P', 'E', 'G'},
}
```



下面这个数组组合字面量

```go
type language struct {
	name string
	year int
}

var _ = [...]language{
	language{"C", 1972},
	language{"Python", 1991},
	language{"Go", 2009},
}
```

可以被简化为

```go
var _ = [...]language{
	{"C", 1972},
	{"Python", 1991},
	{"Go", 2009},
}
```



下面这个映射组合字面量

```go
type LangCategory struct {
	dynamic bool
	strong  bool
}

// 此映射值的类型的键值类型为一个结构体类型，
// 元素类型为另一个映射类型：map[string]int。
var _ = map[LangCategory]map[string]int{
	LangCategory{true, true}: map[string]int{
		"Python": 1991,
		"Erlang": 1986,
	},
	LangCategory{true, false}: map[string]int{
		"JavaScript": 1995,
	},
	LangCategory{false, true}: map[string]int{
		"Go":   2009,
		"Rust": 2010,
	},
	LangCategory{false, false}: map[string]int{
		"C": 1972,
	},
}
```

可以被简化为

```go
var _ = map[LangCategory]map[string]int{
	{true, true}: {
		"Python": 1991,
		"Erlang": 1986,
	},
	{true, false}: {
		"JavaScript": 1995,
	},
	{false, true}: {
		"Go":   2009,
		"Rust": 2010,
	},
	{false, false}: {
		"C": 1972,
	},
}
```

注意，在上面的几个例子中，最后一个元素后的逗号不能被省略。原因详见后面的[断行规则](https://gfw.go101.org/article/line-break-rules.html)一文。



### 容器值的比较

在[Go类型系统概述](https://gfw.go101.org/article/type-system-overview.html#types-not-support-comparison)一文中，我们已经了解到映射和切片类型都属于不可比较类型。 所以任意两个映射值（或切片值）是不能相互比较的。

尽管两个映射值和切片值是不能比较的，但是一个映射值或者切片值可以和预声明的`nil`标识符进行比较以检查此映射值或者切片值是否为一个零值。

大多数数组类型都是可比较类型，除了元素类型为不可比较类型的数组类型。

当比较两个数组值时，它们的对应元素将按照逐一被比较（可以认为按照下标顺序比较）。这两个数组只有在它们的对应元素都相等的情况下才相等；当一对元素被发现不相等的或者[在比较中产生恐慌](https://gfw.go101.org/article/interface.html#comparison)的时候，对数组的比较将提前结束。

一个例子：

```go
package main

import "fmt"

func main() {
	var a [16]byte
	var s []int
	var m map[string]int

	fmt.Println(a == a)   // true
	fmt.Println(m == nil) // true
	fmt.Println(s == nil) // true
	fmt.Println(nil == map[string]int{}) // false
	fmt.Println(nil == []int{})          // false

	// 下面这些行编译不通过。
	/*
	_ = m == m
	_ = s == s
	_ = m == map[string]int(nil)
	_ = s == []int(nil)
	var x [16][]int
	_ = x == x
	var y [16]map[int]bool
	_ = y == y
	*/
}
```





### 查看容器值的长度和容量

除了上面已提到的容器长度属性（此容器中含有有多少个元素），每个容器值还有一个容量属性。 一个数组值的容量总是和它的长度相等；一个非零映射值的容量可以被认为是无限大的。切片值的容量的含义将在后续章节介绍。 一个切片值的容量总是不小于此切片值的长度。在编程中，只有切片值的容量有实际意义。

我们可以调用内置函数`len`来获取一个容器值的长度，或者调用内置函数`cap`来获取一个容器值的容量。 这两个函数都返回一个`int`类型确定结果值。 因为非零映射值的容量是无限大，所以`cap`并不适用于映射值。

一个数组值的长度和容量永不改变。同一个数组类型的所有值的长度和容量都总是和此数组类型的长度相等。 切片值的长度和容量可在运行时刻改变。因为此原因，切片可以被认为是动态数组。 切片在使用上相比数组更为灵活，所以切片（相对数组）在编程用得更为广泛。

一个例子：

```go
package main

import "fmt"

func main() {
	var a [5]int
	fmt.Println(len(a), cap(a)) // 5 5
	var s []int
	fmt.Println(len(s), cap(s)) // 0 0
	s, s2 := []int{2, 3, 5}, []bool{}
	fmt.Println(len(s), cap(s), len(s2), cap(s2)) // 3 3 0 0
	var m map[int]bool
	fmt.Println(len(m)) // 0
	m, m2 := map[int]bool{1: true, 0: false}, map[int]int{}
	fmt.Println(len(m), len(m2)) // 2 0
}
```

上面这个特定的例子中的每个切片值的长度和容量都相等，但这并不是一个普遍定律。 我们将在后面的章节中展示一些长度和容量不相等的切片值。



### 读取和修改容器的元素

一个容器值`v`中存储的对应着键值`k`的元素用语法形式`v[k]`来表示。 今后我们称`v[k]`为一个元素索引表达式。

假设`v`是一个数组或者切片，在`v[k]`中，

- 如果`k`是一个常量，则它必须满足上面列出的[对出现在组合字面量中的索引的要求](https://gfw.go101.org/article/container.html#value-literals)。 另外，如果`v`是一个数组，则`k`必须小于此数组的长度。
- 如果`k`不是一个常量，则它必须为一个整数。 另外它必须为一个非负数并且小于`len(v)`，否则，在运行时刻将产生一个恐慌。
- 如果`v`是一个零值切片，则在运行时刻将产生一个恐慌。

假设`v`是一个映射值，在`v[k]`中，`k`的类型必须为（或者可以隐式转换为）`v`的类型的元素类型。另外，

- 如果`k`是一个动态类型为不可比较类型的接口值，则`v[k]`在运行时刻将造成一个恐慌；
- 如果`v[k]`被用做一个赋值语句中的目标值并且`v`是一个零值nil映射，则`v[k]`在运行时刻将造成一个恐慌；
- 如果`v[k]`用来表示读取映射值`v`中键值`k`对应的元素，则它无论如何都不会产生一个恐慌，即使`v`是一个零值nil映射（假设`k`的估值没有造成恐慌）；
- 如果`v[k]`用来表示读取映射值`v`中键值`k`对应的元素，并且映射值`v`中并不含有对应着键值`k`的条目，则`v[k]`返回一个此映射值的类型的元素类型的零值。 一般情况下，`v[k]`被认为是一个单值表达式。但是在一个`v[k]`被用为唯一源值的赋值语句中，`v[k]`可以返回一个可选的第二个返回值。 此第二个返回值是一个类型不确定布尔值，用来表示是否有对应着键值`k`的条目存储在映射值`v`中。

一个展示了容器元素修改和读取的例子：

```go
package main

import "fmt"

func main() {
	a := [3]int{-1, 0, 1}
	s := []bool{true, false}
	m := map[string]int{"abc": 123, "xyz": 789}
	fmt.Println (a[2], s[1], m["abc"])    // 读取
	a[2], s[1], m["abc"] = 999, true, 567 // 修改
	fmt.Println (a[2], s[1], m["abc"])    // 读取

	n, present := m["hello"]
	fmt.Println(n, present, m["hello"]) // 0 false 0
	n, present = m["abc"]
	fmt.Println(n, present, m["abc"]) // 567 true 567
	m = nil
	fmt.Println(m["abc"]) // 0

	// 下面这两行编译不同过。
	/*
	_ = a[3]  // 下标越界
	_ = s[-1] // 下标越界
	*/

	// 下面这几行每行都会造成一个恐慌。
	_ = a[n]         // panic: 下标越界
	_ = s[n]         // panic: 下标越界
	m["hello"] = 555 // panic: m为一个零值映射
}
```



### 重温一下切片的内部结构

为了更好的理解和解释切片类型和切片值，我们最好对切片的内部结构有一个基本的印象。 在上一篇文章[值部](https://gfw.go101.org/article/value-part.html)中，我们已经了解到官方标准编译器对切片类型的内部定义大致如下：

```go
type _slice struct {
	elements unsafe.Pointer // 引用着底层存储在间接部分上的元素
	len      int            // 长度
	cap      int            // 容量
}
```

虽然其它编译器中切片类型的内部结构可能并不完全和官方标准编译器一致，但应该大体上是相似的。 下面的解释均基于官方标准编译器对切片类型的内部定义。

上面展示的切片的内部定义为切片的直接部分的定义。直接部分的`len`字段表示一个切片当前存储了多少个元素；直接部分的`cap`表示一个切片的容量。 下面这张图描绘了一个切片值的内存布局。

![切片值内存布局](https://gfw.go101.org/article/res/slice-internal.png)



尽管一个切片值的底层元素部分可能位于一个比较大的内存片段上，但是此切片值只能感知到此内存片段上的一个子片段。 比如，上图中的切片值只能感知到灰色的子片段。

在上图中，从下标`len`（包含）到下标`cap`（不包含）对应的元素并不属于图中所示的切片值。 它们只是此切片之中的一些冗余元素槽位，但是它们可能是其它切片（或者数组）值中的有效元素。

下一节将要介绍如何通过调用内置`append`函数来向一个基础切片添加元素而得到一个新的切片。 这个新的结果切片可能和这个基础切片共享起始元素，也可能不共享，具体取决于基础切片的容量（以及长度）和添加的元素数量。

当一个切片被用做一个`append`函数调用中的基础切片，

- 如果添加的元素数量大于此（基础）切片的冗余元素槽位的数量，则一个新的底层内存片段将被开辟出来并用来存放结果切片的元素。 这时，基础切片和结果切片不共享任何底层元素。
- 否则，不会有底层内存片段被开辟出来。这时，基础切片中的所有元素也同时属于结果切片。两个切片的元素都存放于同一个内存片段上。

下下一节将展示一张包含了上述两种情况的图片。

一些其它切片操作也可能会造成两个切片共享底层内存片段的情况。这些操作将在后续章节逐一介绍。

注意，一般我们不能单独修改一个切片值的某个内部字段，除非使用[反射](https://gfw.go101.org/article/container.html#modify-slice-length-and-capacity)或者[非类型安全指针](https://gfw.go101.org/article/unsafe.html)。 换句话说，一般我们只能通过将其它切片赋值给一个切片来同时修改这个切片的三个字段。



### 容器赋值

当一个映射赋值语句执行完毕之后，目标映射值和源映射值将共享底层的元素。 向其中一个映射中添加（或从中删除）元素将体现在另一个映射中。

和映射一样，当一个切片赋值给另一个切片后，它们将共享底层的元素。它们的长度和容量也相等。 但是和映射不同，如果以后其中一个切片改变了长度或者容量，此变化不会体现到另一个切片中。

当一个数组被赋值给另一个数组，所有的元素都将被从源数组复制到目标数组。赋值完成之后，这两个数组不共享任何元素。

一个例子：

```go
package main

import "fmt"

func main() {
	m0 := map[int]int{0:7, 1:8, 2:9}
	m1 := m0
	m1[0] = 2
	fmt.Println(m0, m1) // map[0:2 1:8 2:9] map[0:2 1:8 2:9]

	s0 := []int{7, 8, 9}
	s1 := s0
	s1[0] = 2
	fmt.Println(s0, s1) // [2 8 9] [2 8 9]

	a0 := [...]int{7, 8, 9}
	a1 := a0
	a1[0] = 2
	fmt.Println(a0, a1) // [7 8 9] [2 8 9]
}
```





### 添加和删除容器元素

向一个映射中添加一个条目的语法和修改一个映射元素的语法是一样的。 比如，对于一个非零映射值`m`，如果当前`m`中尚未存储条目`(k, e)`，则下面的语法形式将把此条目存入`m`；否则，下面的语法形式将把键值`k`对应的元素值更新为`e`。

```go
m[k] = e
```



内置函数`delete`用来从一个映射中删除一个条目。比如，下面的`delete`调用将把键值`k`对应的条目从映射`m`中删除。 如果映射`m`中未存储键值为`k`的条目，则此调用为一个空操作，它不会产生一个恐慌，即使`m`是一个nil零值映射。

```go
delete(m, k)
```



下面的例子展示了如何向一个映射添加和从一个映射删除条目。

```go
package main

import "fmt"

func main() {
	m := map[string]int{"Go": 2007}
	m["C"] = 1972     // 添加
	m["Java"] = 1995  // 添加
	fmt.Println(m)    // map[C:1972 Go:2007 Java:1995]
	m["Go"] = 2009    // 修改
	delete(m, "Java") // 删除
	fmt.Println(m)    // map[C:1972 Go:2009]
}
```

注意，在Go 1.12之前，映射打印结果中的条目顺序并不固定，两次打印结果可能并不相同。

一个数组中的元素个数总是恒定的，我们无法向其中添加元素，也无法从其中删除元素。但是可寻址的数组值中的元素是可以被修改的。

我们可以通过调用内置`append`函数，以一个切片为基础，来添加不定数量的元素并返回一个新的切片。 此新的结果切片包含着基础切片中所有的元素和所有被添加的元素。 注意，基础切片并未被此`append`函数调用所修改。 当然，如果我们愿意（事实上在实践中常常如此），我们可以将结果切片赋值给基础切片以修改基础切片。

Go中并未提供一个内置方式来从一个切片中删除一个元素。 我们必须使用`append`函数和后面将要介绍的子切片语法一起来实现元素删除操作。 切片元素的删除和插入将在后面的[更多切片操作](https://gfw.go101.org/article/container.html#slice-manipulations)一节中介绍。 本节仅展示如何使用`append`内置函数。

下面是一个如何使用`append`内置函数的例子。

```go
package main

import "fmt"

func main() {
	s0 := []int{2, 3, 5}
	fmt.Println(s0, cap(s0)) // [2 3 5] 3
	s1 := append(s0, 7)      // 添加一个元素
	fmt.Println(s1, cap(s1)) // [2 3 5 7] 6
	s2 := append(s1, 11, 13) // 添加两个元素
	fmt.Println(s2, cap(s2)) // [2 3 5 7 11 13] 6
	s3 := append(s0)         // <=> s3 := s0
	fmt.Println(s3, cap(s3)) // [2 3 5] 3
	s4 := append(s0, s0...)  // 以s0为基础添加s0中所有的元素
	fmt.Println(s4, cap(s4)) // [2 3 5 2 3 5] 6

	s0[0], s1[0] = 99, 789
	fmt.Println(s2[0], s3[0], s4[0]) // 789 99 2
}
```



注意，内置`append`函数是一个[变长参数函数](https://gfw.go101.org/article/function.html#variadic-function)（下下篇文章中介绍）。 它有两个参数，其中第二个参数（形参）为一个[变长参数](https://gfw.go101.org/article/function.html#variadic-parameter)。

变长参数函数将在下下篇文章中解释。目前，我们只需知道变长参数函数调用中的实参有两种传递方式。 在上面的例子中，第*8*行、第*10*行和第*12*行使用了同一种方式，第*14*行使用了另外一种方式。 在第一种方式中，零个或多个实参元素值可以传递给`append`函数的第二个形参。 在第二种方式中，一个（和第一个实参同元素类型的）实参切片传递给了第二个形参，此切片实参必须跟随三个点`...`。 关于变长参数函数调用，详见[下下篇文章](https://gfw.go101.org/article/function.html#variadic-call)。

在上例中，第*14*行等价于

```go
	s4 := append(s0, s0[0], s0[1], s0[2])
```



第*8*行等价于

```go
	s1 := append(s0, []int{7}...)
```



第*10*行等价于

```go
	s2 := append(s1, []int{11, 13}...)
```



对于三个点方式，`append`函数并不要求第二个实参的类型和第一个实参一致，但是它们的元素类型必须一致。 换句话说，它们的[底层类型](https://gfw.go101.org/article/type-system-overview.html#underlying-type)必须一致。

在上面的程序中，

- 第*8*行的`append`函数调用将为结果切片`s1`开辟一段新的内存。 原因是切片`s0`中没有足够的冗余元素槽位来容纳新添加的元素。 第*14*行的`append`函数调用也是同样的情况。
- 第*10*行的`append`函数调用不会为结果切片`s2`开辟新的内存片段。 原因是切片`s1`中的冗余元素槽位足够容纳新添加的元素。

所以，上面的程序中在退出之前，切片`s1`和`s2`共享一些元素，切片`s0`和`s3`共享所有的元素。 下面这张图描绘了在上面的程序结束之前各个切片的状态。

![各个切片状态](https://gfw.go101.org/article/res/slice-append.png)



请注意，当一个`append`函数调用需要为结果切片开辟内存时，结果切片的容量取决于具体编译器实现。 在这种情况下，对于官方标准编译器，如果基础切片的容量较小，则结果切片的容量至少为基础切片的两倍。 这样做的目的是使结果切片有足够多的冗余元素槽位，以防止此结果切片被用做后续其它`append`函数调用的基础切片时再次开辟内存。

上面提到了，在实际编程中，我们常常将`append`函数调用的结果赋值给基础切片。 比如：

```go
package main

import "fmt"

func main() {
	var s = append([]string(nil), "array", "slice")
	fmt.Println(s)      // [array slice]
	fmt.Println(cap(s)) // 2
	s = append(s, "map")
	fmt.Println(s)      // [array slice map]
	fmt.Println(cap(s)) // 4
	s = append(s, "channel")
	fmt.Println(s)      // [array slice map channel]
	fmt.Println(cap(s)) // 4
}
```



截至目前（Go 1.16），`append`函数调用的第一个实参不能为类型不确定的`nil`。



### 使用内置`make`函数来创建切片和映射

除了使用组合字面量来创建映射和切片，我们还可以使用内置`make`函数来创建映射和切片。 数组不能使用内置`make`函数来创建。

顺便说一句，内置`make`函数也可以用来创建以后将要介绍的[通道](https://gfw.go101.org/article/channel.html)值。

假设`M`是一个映射类型并且`n`是一个整数，我们可以用下面的两种函数调用来各自生成一个类型为`M`的映射值。

```go
make(M, n)
make(M)
```

第一个函数调用形式创建了一个可以容纳至少`n`个条目而无需再次开辟内存的空映射值。 第二个函数调用形式创建了一个可以容纳一个小数目的条目而无需再次开辟内存的空映射值。此小数目的值取决于具体编译器实现。

注意：第二个参数`n`可以为负或者零，这时对应的调用将被视为上述第二种调用形式。

假设`S`是一个切片类型，`length`和`capacity`是两个非负整数，并且`length`小于等于`capacity`，我们可以用下面的两种函数调用来各自生成一个类型为`S`的切片值。`length`和`capacity`的类型必须均为整数类型（两者可以不一致）。

```go
make(S, length, capacity)
make(S, length) // <=> make(S, length, length)
```

第一个函数调用创建了一个长度为`length`并且容量为`capacity`的切片。 第二个函数调用创建了一个长度为`length`并且容量也为`length`的切片。

使用`make`函数创建的切片中的所有元素值均被初始化为（结果切片的元素类型的）零值。

下面是一个展示了如何使用`make`函数来创建映射和切片的例子：

```go
package main

import "fmt"

func main() {
	// 创建映射。
	fmt.Println(make(map[string]int)) // map[]
	m := make(map[string]int, 3)
	fmt.Println(m, len(m)) // map[] 0
	m["C"] = 1972
	m["Go"] = 2009
	fmt.Println(m, len(m)) // map[C:1972 Go:2009] 2

	// 创建切片。
	s := make([]int, 3, 5)
	fmt.Println(s, len(s), cap(s)) // [0 0 0] 3 5
	s = make([]int, 2)
	fmt.Println(s, len(s), cap(s)) // [0 0] 2 2
}
```





### 使用内置`new`函数来创建容器值

在前面的[指针](https://gfw.go101.org/article/pointer.html)一文中，我们已经了解到内置`new`函数可以用来为一个任何类型的值开辟内存并返回一个存储有此值的地址的指针。 用`new`函数开辟出来的值均为零值。因为这个原因，`new`函数对于创建映射和切片值来说没有任何价值。

使用`new`函数来用来创建数组值并非是完全没有意义的，但是在实践中很少这么做，因为使用组合字面量来创建数组值更为方便。

一个使用`new`函数创建容器值的例子：

```go
package main

import "fmt"

func main() {
	m := *new(map[string]int)   // <=> var m map[string]int
	fmt.Println(m == nil)       // true
	s := *new([]int)            // <=> var s []int
	fmt.Println(s == nil)       // true
	a := *new([5]bool)          // <=> var a [5]bool
	fmt.Println(a == [5]bool{}) // true
}
```





### 容器元素的可寻址性

一些关于容器元素的可寻址性的事实：

- 如果一个数组是可寻址的，则它的元素也是可寻址的；反之亦然，即如果一个数组是不可寻址的，则它的元素也是不可寻址的。 原因很简单，因为一个数组只含有一个（直接）[值部](https://gfw.go101.org/article/value-part.html)，并且它的所有元素和此直接值部均承载在同一个[内存块](https://gfw.go101.org/article/memory-block.html)上。
- 一个切片值的任何元素都是可寻址的，即使此切片本身是不可寻址的。 这是因为一个切片的底层元素总是存储在一个被开辟出来的内存片段（间接值部）上。
- 任何映射元素都是不可寻址的。原因详见[此条问答](https://gfw.go101.org/article/unofficial-faq.html#maps-are-unaddressable)。

一个例子：

```go
package main

import "fmt"

func main() {
	a := [5]int{2, 3, 5, 7}
	s := make([]bool, 2)
	pa2, ps1 := &a[2], &s[1]
	fmt.Println(*pa2, *ps1) // 5 false
	a[2], s[1] = 99, true
	fmt.Println(*pa2, *ps1) // 99 true
	ps0 := &[]string{"Go", "C"}[0]
	fmt.Println(*ps0) // Go

	m := map[int]bool{1: true}
	_ = m
	// 下面这几行编译不通过。
	/*
	_ = &[3]int{2, 3, 5}[0]
	_ = &map[int]bool{1: true}[1]
	_ = &m[1]
	*/
}
```



一般来说，一个不可寻址的值的直接部分是不可修改的。但是映射元素是个例外。 映射元素虽然不可寻址，但是每个映射元素可以被整个修改（但不能被部分修改）。 对于大多数做为映射元素类型的类型，在修改它们的值的时候，一般体现不出来整个修改和部分修改的差异。 但是如果一个映射的元素类型为数组或者结构体类型，这个差异是很明显的。

在上一篇文章[值部](https://gfw.go101.org/article/value-part.html)中，我们了解到每个数组或者结构体值都是仅含有一个直接部分。所以

- 如果一个映射类型的元素类型为一个结构体类型，则我们无法修改此映射类型的值中的每个结构体元素的单个字段。 我们必须整体地同时修改所有结构体字段。
- 如果一个映射类型的元素类型为一个数组类型，则我们无法修改此映射类型的值中的每个数组元素的单个元素。 我们必须整体地同时修改所有数组元素。

一个例子：

```go
package main

import "fmt"

func main() {
	type T struct{age int}
	mt := map[string]T{}
	mt["John"] = T{age: 29} // 整体修改是允许的
	ma := map[int][5]int{}
	ma[1] = [5]int{1: 789} // 整体修改是允许的

	// 这两个赋值编译不通过，因为部分修改一个映射
	// 元素是非法的。这看上去确实有些反直觉。
	/*
	ma[1][1] = 123      // error
	mt["John"].age = 30 // error
	*/

	// 读取映射元素的元素或者字段是没问题的。
	fmt.Println(ma[1][1])       // 789
	fmt.Println(mt["John"].age) // 29
}
```



为了让上例中的两行编译不通过的两行赋值语句编译通过，欲修改的映射元素必须先存放在一个临时变量中，然后修改这个临时变量，最后再用这个临时变量整体覆盖欲修改的映射元素。比如：

```go
package main

import "fmt"

func main() {
	type T struct{age int}
	mt := map[string]T{}
	mt["John"] = T{age: 29}
	ma := map[int][5]int{}
	ma[1] = [5]int{1: 789}

	t := mt["John"] // 临时变量
	t.age = 30
	mt["John"] = t // 整体修改

	a := ma[1] // 临时变量
	a[1] = 123
	ma[1] = a // 整体修改

	fmt.Println(ma[1][1], mt["John"].age) // 123 30
}
```

注意：刚提到的这个限制[m可能会在以后被移除](https://github.com/golang/go/issues/3117)。



### 从数组或者切片派生切片（取子切片）

我们可以从一个基础切片或者一个可寻址的基础数组派生出另一个切片。此派生操作也常称为一个取子切片操作。 派生出来的切片的元素和基础切片（或者数组）的元素位于同一个内存片段上。或者说，派生出来的切片和基础切片（或者数组）将共享一些元素。

Go中有两种取子切片的语法形式（假设`baseContainer`是一个切片或者数组）：

```go
baseContainer[low : high]       // 双下标形式
baseContainer[low : high : max] // 三下标形式
```

上面所示的双下标形式等价于下面的三下标形式：

```go
baseContainer[low : high : cap(baseContainer)]
```

所以双下标形式是三下标形式的特例。在实践中，双下标形式使用得相对更为广泛。

（注意：三下标形式是从Go 1.2开始支持的。）

上面所示的取子切片表达式的语法形式中的下标必须满足下列关系，否则代码要么编译不通过，要么在运行时刻将造成恐慌。

```go
// 双下标形式
0 <= low <= high <= cap(baseContainer)

// 三下标形式
0 <= low <= high <= max <= cap(baseContainer)
```

不满足上述关系的取子切片表达式要么编译不通过，要么在运行时刻将导致一个恐慌。

注意：

- 只要上述关系均满足，下标`low`和`high`都可以大于`len(baseContainer)`。但是它们一定不能大于`cap(baseContainer)`。
- 如果`baseContainer`是一个零值nil切片，只要上面所示的子切片表达式中下标的值均为`0`，则这两个子切片表达式不会造成恐慌。 在这种情况下，结果切片也是一个nil切片。

子切片表达式的结果切片的长度为`high - low`、容量为`max - low`。 派生出来的结果切片的长度可能大于基础切片的长度，但结果切片的容量绝不可能大于基础切片的容量。

在实践中，我们常常在子切片表达式中省略若干下标，以使代码看上去更加简洁。省略规则如下：

- 如果下标`low`为零，则它可被省略。此条规则同时适用于双下标形式和三下标形式。
- 如果下标`high`等于`len(baseContainer)`，则它可被省略。此条规则同时只适用于双下标形式。
- 三下标形式中的下标`max`在任何情况下都不可被省略。

比如，下面的子切片表达式都是相互等价的：

```go
baseContainer[0 : len(baseContainer)]
baseContainer[: len(baseContainer)]
baseContainer[0 :]
baseContainer[:]
baseContainer[0 : len(baseContainer) : cap(baseContainer)]
baseContainer[: len(baseContainer) : cap(baseContainer)]
```

一个使用了子切片语法的例子：

```go
package main

import "fmt"

func main() {
	a := [...]int{0, 1, 2, 3, 4, 5, 6}
	s0 := a[:]     // <=> s0 := a[0:7:7]
	s1 := s0[:]    // <=> s1 := s0
	s2 := s1[1:3]  // <=> s2 := a[1:3]
	s3 := s1[3:]   // <=> s3 := s1[3:7]
	s4 := s0[3:5]  // <=> s4 := s0[3:5:7]
	s5 := s4[:2:2] // <=> s5 := s0[3:5:5]
	s6 := append(s4, 77)
	s7 := append(s5, 88)
	s8 := append(s7, 66)
	s3[1] = 99
	fmt.Println(len(s2), cap(s2), s2) // 2 6 [1 2]
	fmt.Println(len(s3), cap(s3), s3) // 4 4 [3 99 77 6]
	fmt.Println(len(s4), cap(s4), s4) // 2 4 [3 99]
	fmt.Println(len(s5), cap(s5), s5) // 2 2 [3 99]
	fmt.Println(len(s6), cap(s6), s6) // 3 4 [3 99 77]
	fmt.Println(len(s7), cap(s7), s7) // 3 4 [3 4 88]
	fmt.Println(len(s8), cap(s8), s8) // 4 4 [3 4 88 66]
}
```



下面这张图描绘了上面的程序在退出之前各个数组和切片的状态。

![数字和切片状态](https://gfw.go101.org/article/res/slice-subslice-2.png)

从这张图片可以看出，切片`s7`和`s8`共享存储它们的元素的底层内存片段，其它切片和数组`a`共享同一个存储元素的内存片段。

请注意，子切片操作有可能会造成暂时性的内存泄露。 比如，下面在这个函数中开辟的内存块中的前50个元素槽位在它的调用返回之后将不再可见。 这50个元素槽位所占内存浪费了，这属于暂时性的内存泄露。 当这个函数中开辟的内存块今后不再被任何切片所引用，此内存块将被回收，这时内存才不再继续泄漏。

```go
func f() []int {
	s := make([]int, 10, 100)
	return s[50:60]
}
```

请注意，在上面这个函数中，子切片表达式中的起始下标（`50`）比`s`的长度（`10`）要大，这是允许的。



### 切片转化为数组指针

从Go 1.17开始，一个切片可以被转化为一个相同元素类型的数组的指针类型。 但是如果数组的长度大于被转化切片的长度，则将导致恐慌产生。一个例子：

```go
package main

type S []int
type A [2]int
type P *A

func main() {
	var x []int
	var y = make([]int, 0)
	var x0 = (*[0]int)(x) // okay, x0 == nil
	var y0 = (*[0]int)(y) // okay, y0 != nil
	_, _ = x0, y0

	var z = make([]int, 3, 5)
	var _ = (*[3]int)(z) // okay
	var _ = (*[2]int)(z) // okay
	var _ = (*A)(z)      // okay
	var _ = P(z)         // okay

	var w = S(z)
	var _ = (*[3]int)(w) // okay
	var _ = (*[2]int)(w) // okay
	var _ = (*A)(w)      // okay
	var _ = P(w)         // okay

	var _ = (*[4]int)(z) // 会产生恐慌
}
```



### 使用内置`copy`函数来复制切片元素

我们可以使用内置`copy`函数来将一个切片中的元素复制到另一个切片。 这两个切片的类型可以不同，但是它们的元素类型必须相同。 换句话说，这两个切片的类型的底层类型必须相同。 `copy`函数的第一个参数为目标切片，第二个参数为源切片。 传递给一个`copy`函数调用的两个实参可以共享一些底层元素。 `copy`函数返回复制了多少个元素，此值（`int`类型）为这两个切片的长度的较小值。

结合上一节介绍的子切片语法，我们可以使用`copy`函数来在两个数组之间或者一个数组与一个切片之间复制元素。

一个例子：

```go
package main

import "fmt"

func main() {
	type Ta []int
	type Tb []int
	dest := Ta{1, 2, 3}
	src := Tb{5, 6, 7, 8, 9}
	n := copy(dest, src)
	fmt.Println(n, dest) // 3 [5 6 7]
	n = copy(dest[1:], dest)
	fmt.Println(n, dest) // 2 [5 5 6]

	a := [4]int{} // 一个数组
	n = copy(a[:], src)
	fmt.Println(n, a) // 4 [5 6 7 8]
	n = copy(a[:], a[2:])
	fmt.Println(n, a) // 2 [7 8 7 8]
}
```



事实上，`copy`并不是一个基本函数。我们可以用`append`来实现它。

```go
// 假设元素类型为T。
func Copy(dest, src []T) int {
	if len(dest) < len(src) {
		_ = append(dest[:0], src[:len(dest)]...)
		return len(dest)
	} else {
		_ = append(dest[:0], src...)
		return len(src)
	}
}
```

尽管`copy`函数不是一个基本函数，它比上面的用`append`的实现使用起来要方便得多。

从另外一个角度，我们也可以认为`append`不是一个基本函数，因为我们可以用`make`加`copy`函数来实现它。

注意，做为一个特例，`copy`函数可以用来[将一个字符串中的字节复制到一个字节切片](https://gfw.go101.org/article/string.html#use-string-as-byte-slice)。

截至目前（Go 1.16），`copy`函数调用的两个实参均不能为类型不确定的`nil`。



### 遍历容器元素

在Go中，我们可以使用下面的语法形式来遍历一个容器中的键值和元素：

```go
for key, element = range aContainer {
	// 使用key和element ...
}
```

在此语法形式中，`for`和`range`为两个关键字，`key`和`element`称为循环变量。 如果`aContainer`是一个切片或者数组（或者数组指针，见后），则`key`的类型必须为内置类型`int`。

上面所示的`for-range`语法形式中的等号`=`也可以是一个变量短声明符号`:=`。 当短声明符号被使用的时候，`key`和`element`总是两个新声明的变量，这时如果`aContainer`是一个切片或者数组（或者数组指针），则`key`的类型被推断为内置类型`int`。

和传统的`for`循环流程控制一样，每个`for-range`循环流程控制形成了两个代码块，其中一个是隐式的，另一个是显式的（花括号`之间`的部分）。 此显式的代码块内嵌在隐式的代码块之中。

和`for`循环流程控制一样，`break`和`continue`也可以使用在一个`for-range`循环流程控制中的显式代码块中。

一个例子：

```go
package main

import "fmt"

func main() {
	m := map[string]int{"C": 1972, "C++": 1983, "Go": 2009}
	for lang, year := range m {
		fmt.Printf("%v: %v \n", lang, year)
	}

	a := [...]int{2, 3, 5, 7, 11}
	for i, prime := range a {
		fmt.Printf("%v: %v \n", i, prime)
	}

	s := []string{"go", "defer", "goto", "var"}
	for i, keyword := range s {
		fmt.Printf("%v: %v \n", i, keyword)
	}
}
```



`for-range`循环代码块有一些变种形式：

```go
// 忽略键值循环变量。
for _, element = range aContainer {
	// ...
}

// 忽略元素循环变量。
for key, _ = range aContainer {
	element = aContainer[key]
	// ...
}

// 舍弃元素循环变量。此形式和上一个变种等价。
for key = range aContainer {
	element = aContainer[key]
	// ...
}

// 键值和元素循环变量均被忽略。
for _, _ = range aContainer {
	// 这个变种形式没有太大实用价值。
}

// 键值和元素循环变量均被舍弃。此形式和上一个变种等价。
for range aContainer {
	// 这个变种形式没有太大实用价值。
}
```

遍历一个nil映射或者nil切片是允许的。这样的遍历可以看作是一个空操作。

一些关于遍历映射条目的细节：

- 映射中的条目的遍历顺序是不确定的（可以认为是随机的）。或者说，同一个映射中的条目的两次遍历中，条目的顺序很可能是不一致的，即使在这两次遍历之间，此映射并未发生任何改变。
- 如果在一个映射中的条目的遍历过程中，一个还没有被遍历到的条目被删除了，则此条目保证不会被遍历出来。
- 如果在一个映射中的条目的遍历过程中，一个新的条目被添加入此映射，则此条目并不保证将在此遍历过程中被遍历出来。

如果可以确保没有其它协程操纵一个映射`m`，则下面的代码保证将清空`m`中所有条目。

```go
for key := range m {
	delete(m, key)
}
```



当然，数组和切片元素也可以用传统的`for`循环来遍历。

```go
for i := 0; i < len(anArrayOrSlice); i++ {
	element := anArrayOrSlice[i]
	// ...
}
```



对一个`for-range`循环代码块

```go
for key, element = range aContainer {...}
```

有三个重要的事实存在：

1. 被遍历的容器值是

   ```
   aContainer
   ```

   的

   一个副本

   。 注意，

   只有`aContainer`的直接部分被复制了

   。 此副本是一个匿名的值，所以它是不可被修改的。

   - 如果`aContainer`是一个数组，那么在遍历过程中对此数组元素的修改不会体现到循环变量中。 原因是此数组的副本（被真正遍历的容器）和此数组不共享任何元素。
   - 如果`aContainer`是一个切片（或者映射），那么在遍历过程中对此切片（或者映射）元素的修改将体现到循环变量中。 原因是此切片（或者映射）的副本和此切片（或者映射）共享元素（或条目）。

2. 在遍历中的每个循环步，`aContainer`副本中的一个键值元素对将被赋值（复制）给循环变量。 所以对循环变量的直接部分的修改将不会体现在`aContainer`中的对应元素中。 （因为这个原因，并且`for-range`循环是遍历映射条目的唯一途径，所以最好不要使用大尺寸的映射键值和元素类型，以避免较大的复制负担。）

3. 所有被遍历的键值对将被赋值给**同一对**循环变量实例。

下面这个例子验证了上述第一个和第二个事实。

```go
package main

import "fmt"

func main() {
	type Person struct {
		name string
		age  int
	}
	persons := [2]Person {{"Alice", 28}, {"Bob", 25}}
	for i, p := range persons {
		fmt.Println(i, p)
		// 此修改将不会体现在这个遍历过程中，
		// 因为被遍历的数组是persons的一个副本。
		persons[1].name = "Jack"
		// 此修改不会反映到persons数组中，因为p
		// 是persons数组的副本中的一个元素的副本。
		p.age = 31
	}
	fmt.Println("persons:", &persons)
}
```

输出结果：

```
0 {Alice 28}
1 {Bob 25}
persons: &[{Alice 28} {Jack 25}]
```



如果我们将上例中的数组改为一个切片，则在循环中对此切片的修改将在循环过程中体现出来。 但是对循环变量的修改仍然不会体现在此切片中。

```go
...

	// 改为一个切片。
	persons := []Person {{"Alice", 28}, {"Bob", 25}}
	for i, p := range persons {
		fmt.Println(i, p)
		// 这次，此修改将反映在此次遍历过程中。
		persons[1].name = "Jack"
		// 这个修改仍然不会体现在persons切片容器中。
		p.age = 31
	}
	fmt.Println("persons:", &persons)
}
```

输出结果变成了：

```
0 {Alice 28}
1 {Jack 25}
persons: &[{Alice 28} {Jack 25}]
```



下面这个例子验证了上述的第二个和第三个事实：

```go
package main

import "fmt"

func main() {
	langs := map[struct{ dynamic, strong bool }]map[string]int{
		{true, false}:  {"JavaScript": 1995},
		{false, true}:  {"Go": 2009},
		{false, false}: {"C": 1972},
	}
	// 此映射的键值和元素类型均为指针类型。
	// 这有些不寻常，只是为了讲解目的。
	m0 := map[*struct{ dynamic, strong bool }]*map[string]int{}
	for category, langInfo := range langs {
		m0[&category] = &langInfo
		// 下面这行修改对映射langs没有任何影响。
		category.dynamic, category.strong = true, true
	}
	for category, langInfo := range langs {
		fmt.Println(category, langInfo)
	}

	m1 := map[struct{ dynamic, strong bool }]map[string]int{}
	for category, langInfo := range m0 {
		m1[*category] = *langInfo
	}
	// 映射m0和m1中均只有一个条目。
	fmt.Println(len(m0), len(m1)) // 1 1
	fmt.Println(m1) // map[{true true}:map[C:1972]]
}
```

上面已经提到了，映射条目的遍历顺序是随机的。所以下面前三行的输出顺序可能会略有不同：

```
{false true} map[Go:2009]
{false false} map[C:1972]
{true false} map[JavaScript:1995]
1 1
map[{true true}:map[Go:2009]]
```



复制一个切片或者映射的代价很小，但是复制一个大尺寸的数组的代价比较大。 所以，一般来说，`range`关键字后跟随一个大尺寸数组不是一个好主意。 如果我们要遍历一个大尺寸数组中的元素，我们以遍历从此数组派生出来的一个切片，或者遍历一个指向此数组的指针（详见下一节）。

对于一个数组或者切片，如果它的元素类型的尺寸较大，则一般来说，用第二个循环变量来存储每个循环步中被遍历的元素不是一个好主意。 对于这样的数组或者切片，我们最好忽略或者舍弃`for-range`代码块中的第二个循环变量，或者使用传统的`for`循环来遍历元素。 比如，在下面这个例子中，函数`fa`中的循环效率比函数`fb`中的循环低得多。

```go
type Buffer struct {
	start, end int
	data       [1024]byte
}

func fa(buffers []Buffer) int {
	numUnreads := 0
	for _, buf := range buffers {
		numUnreads += buf.end - buf.start
	}
	return numUnreads
}

func fb(buffers []Buffer) int {
	numUnreads := 0
	for i := range buffers {
		numUnreads += buffers[i].end - buffers[i].start
	}
	return numUnreads
}
```





### 把数组指针当做数组来使用

对于某些情形，我们可以把数组指针当做数组来使用。

我们可以通过在`range`关键字后跟随一个数组的指针来遍历此数组中的元素。 对于大尺寸的数组，这种方法比较高效，因为复制一个指针比复制一个大尺寸数组的代价低得多。 下面的例子中的两个循环是等价的，它们的效率也基本相同。

```go
package main

import "fmt"

func main() {
	var a [100]int

	for i, n := range &a { // 复制一个指针的开销很小
		fmt.Println(i, n)
	}

	for i, n := range a[:] { // 复制一个切片的开销很小
		fmt.Println(i, n)
	}
}
```



如果一个`for-range`循环中的第二个循环变量既没有被忽略，也没有被舍弃，并且`range`关键字后跟随一个nil数组指针，则此循环将造成一个恐慌。 在下面这个例子中，前两个循环都将打印出5个下标，但最后一个循环将导致一个恐慌。

```go
package main

import "fmt"

func main() {
	var p *[5]int // nil

	for i, _ := range p { // okay
		fmt.Println(i)
	}

	for i := range p { // okay
		fmt.Println(i)
	}

	for i, n := range p { // panic
		fmt.Println(i, n)
	}
}
```



我们可以通过数组的指针来访问和修改此数组中的元素。如果此指针是一个nil指针，将导致一个恐慌。

```go
package main

import "fmt"

func main() {
	a := [5]int{2, 3, 5, 7, 11}
	p := &a
	p[0], p[1] = 17, 19
	fmt.Println(a) // [17 19 5 7 11]
	p = nil
	_ = p[0] // panic
}
```



我们可以从一个数组的指针派生出一个切片。从一个nil数组指针派生切片将导致一个恐慌。

```go
package main

import "fmt"

func main() {
	pa := &[5]int{2, 3, 5, 7, 11}
	s := pa[1:3]
	fmt.Println(s) // [3 5]
	pa = nil
	s = pa[0:0] // panic
	// 如果下一行能被执行到，则它也会产生恐慌。
	_ = (*[0]byte)(nil)[:]
}
```



内置`len`和`cap`函数调用接受数组指针做为实参。 nil数组指针实参不会导致恐慌。

```go
var pa *[5]int // == nil
fmt.Println(len(pa), cap(pa)) // 5 5
```





### `memclr`优化

假设`t0`是一个类型`T`的零值字面量，并且`a`是一个元素类型为`T`的数组或者切片，则官方标准编译器将把下面的单循环变量`for-range`代码块优化为一个[内部的`memclr`调用](https://github.com/golang/go/issues/5373)。 大多数情况下，此`memclr`调用比一个一个地重置元素要快。

```go
for i := range a {
	a[i] = t0
}
```

此优化在官方标准编译器1.5版本中被引入。

此优化不适用于`a`为一个数组指针的情形，至少到目前（Go 1.16）为止是这样。 所以，如果你打算重置一个数组，最好不要在`range`关键字后跟随此数组的指针。



### 内置函数`len`和`cap`的调用可能会在编译时刻被估值

如果传递给内置函数`len`或者`cap`的一个调用的实参是一个数组或者数组指针，则此调用将在编译时刻被估值。 此估值结果是一个类型为内置类型`int`的类型确定常量值。

一个例子：

```go
package main

import "fmt"

var a [5]int
var p *[7]string

// N和M都是类型为int的类型确定值。
const N = len(a)
const M = cap(p)

func main() {
	fmt.Println(N) // 5
	fmt.Println(M) // 7
}
```





### 单独修改一个切片的长度或者容量

上面已经提到了，一般来说，一个切片的长度和容量不能被单独修改。一个切片只有通过赋值的方式被整体修改。 但是，事实上，我们可以通过反射的途径来单独修改一个切片的长度或者容量。 反射将在[后面的一篇文章](https://gfw.go101.org/article/reflection.html)中详解。

一个例子：

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	s := make([]int, 2, 6)
	fmt.Println(len(s), cap(s)) // 2 6

	reflect.ValueOf(&s).Elem().SetLen(3)
	fmt.Println(len(s), cap(s)) // 3 6

	reflect.ValueOf(&s).Elem().SetCap(5)
	fmt.Println(len(s), cap(s)) // 3 5
}
```

传递给函数`reflect.SetLen`调用的第二个实参值必须不大于第一个实参切片值的容量。 传递给函数`reflect.SetCap`调用的第二个实参值必须不小于第一个实参切片值的长度并且须不大于第一个实参切片值的容量。 否则，在运行时刻将产生一个恐慌。

此反射方法的效率很低，远低于一个切片的赋值。



### 更多切片操作

Go不支持更多的内置切片操作，比如切片克隆、元素删除和插入。 我们必须用上面提到的各种内置操作来实现这些操作。

在下面当前大节中的例子中，假设`s`是被谈到的切片、`T`是它的元素类型、`t0`是类型`T`的零值字面量。

#### 切片克隆

对于目前的标准编译器（1.16版本），最简单的克隆一个切片的方法为：

```go
sClone := append(s[:0:0], s...)
```

我们也可以使用下面这种实现。但是和上面这个实现相比，它有一个不完美之处：如果源切片`s`是一个空切片（但是非nil），则结果切片是一个nil切片。

```go
sClone := append([]T(nil), s...)
```

上面这两种append实现都有一个缺点：它们开辟的内存块常常会比需要的略大一些从而可能造成一点小小的不必要的性能损失。 我们可以使用这两种方法来避免这个缺点：

```go
// 两行make+copy实现：
sClone := make([]T, len(s))
copy(sClone, s)

// 或者下面的make+append实现。
// 对于目前的官方Go工具链v1.16来说，这种
// 实现比上面的make+copy实现略慢一点。
sClone := append(make([]T, 0, len(s)), s...)
```



上面这两种make方法都有一个缺点：如果`s`是一个nil切片，则使用此方法将得到一个非nil切片。 不过，在编程实践中，我们常常并不需要追求克隆的完美性。如果我们确实需要，则需要多写几行：

```go
var sClone []T
if s != nil {
	sClone = make([]T, len(s))
	copy(sClone, s)
}
```

在Go官方工具链1.15版本之前，对于一些常见的使用场景，使用`append`来克隆切片比使用`make`加`copy`要[高效得多](https://github.com/go101/go101/wiki/How-to-efficiently-clone-a-slice%3F)。但是从1.15版本开始，官方标准编译器对`make+copy`这种方法做了特殊的优化，从而使得此方法总是比使用`append`来克隆切片高效。



#### 删除一段切片元素

前面已经提到了切片的元素在内存中是连续存储的，相邻元素之间是没有间隙的。所以，当切片的一个元素段被删除时，

- 如果剩余元素的次序必须保持原样，则被删除的元素段后面的每个元素都得前移。
- 如果剩余元素的次序不需要保持原样，则我们可以将尾部的一些元素移到被删除的元素的位置上。

在下面的例子中，假设`from`（包括）和`to`（不包括）是两个合法的下标，并且`from`不大于`to`。

```go
// 第一种方法（保持剩余元素的次序）：
s = append(s[:from], s[to:]...)

// 第二种方法（保持剩余元素的次序）：
s = s[:from + copy(s[from:], s[to:])]

// 第三种方法（不保持剩余元素的次序）：
if n := to-from; len(s)-to < n {
	copy(s[from:to], s[to:])
} else {
	copy(s[from:to], s[len(s)-n:])
}
s = s[:len(s)-(to-from)]
```

如果切片的元素可能引用着其它值，则我们应该重置因为删除元素而多出来的元素槽位上的元素值，以避免暂时性的内存泄露：

```go
// "len(s)+to-from"是删除操作之前切片s的长度。
temp := s[len(s):len(s)+to-from]
for i := range temp {
	temp[i] = t0
}
```

前面已经提到了，上面这个`for-range`循环将被官方标准编译器优化为一个`memclr`调用。



#### 删除一个元素

删除一个元素是删除一个元素段的特例。在实现上可以简化一些。

在下面的例子中，假设`i`将被删除的元素的下标，并且它是一个合法的下标。

```go
// 第一种方法（保持剩余元素的次序）：
s = append(s[:i], s[i+1:]...)

// 第二种方法（保持剩余元素的次序）：
s = s[:i + copy(s[i:], s[i+1:])]

// 上面两种方法都需要复制len(s)-i-1个元素。

// 第三种方法（不保持剩余元素的次序）：
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```



如果切片的元素可能引用着其它值，则我们应该重置刚多出来的元素槽位上的元素值，以避免暂时性的内存泄露：

```go
s[len(s):len(s)+1][0] = t0
// 或者
s[:len(s)+1][len(s)] = t0
```



#### 条件性地删除切片元素

有时，我们需要删除满足某些条件的切片元素。

```go
// 假设T是一个小尺寸类型。
func DeleteElements(s []T, keep func(T) bool, clear bool) []T {
	// result := make([]T, 0, len(s))
	result := s[:0] // 无须开辟内存
	for _, v := range s {
		if keep(v) {
			result = append(result, v)
		}
	}
	if clear { // 避免暂时性的内存泄露。
		temp := s[len(result):]
		for i := range temp {
			temp[i] = t0 // t0是类型T的零值
		}
	}
	return result
}
```

注意：如果`T`是一个大尺寸类型，请[慎用](https://gfw.go101.org/article/value-copy-cost.html#copy-costs)`T`做为参数类型和使用双循环变量`for-range`代码块遍历元素类型为`T`的切片。

#### 将一个切片中的所有元素插入到另一个切片中

假设插入位置`i`是一个合法的下标并且切片`elements`中的元素将被插入到另一个切片`s`中。

```go
// 第一种方法:单行实现。
s = append(s[:i], append(elements, s[i:]...)...)

// 上面这种单行实现把s[i:]中的元素复制了两次，并且它可能
// 最多导致两次内存开辟（最少一次）。
// 下面这种繁琐的实现只把s[i:]中的元素复制了一次，并且
// 它最多只会导致一次内存开辟（最少零次）。
// 但是，在当前的官方标准编译器实现中（1.16版本），此
// 繁琐实现中的make调用将会把所有刚开辟出来的元素清零。
// 这其实是没有必要的。所以此繁琐实现并非总是比上面的
// 单行实现效率更高。事实上，它仅在处理小切片时更高效。

if cap(s) >= len(s) + len(elements) {
	s = s[:len(s)+len(elements)]
	copy(s[i+len(elements):], s[i:])
	copy(s[i:], elements)
} else {
	x := make([]T, 0, len(elements)+len(s))
	x = append(x, s[:i]...)
	x = append(x, elements...)
	x = append(x, s[i:]...)
	s = x
}

// Push（插入到结尾）。
s = append(s, elements...)

// Unshift（插入到开头）。
s = append(elements, s...)
```



#### 插入若干独立的元素

插入若干独立的元素和插入一个切片中的所有元素类似。 我们可以使用切片组合字面量构建一个临时切片，然后使用上面的方法插入这些元素。

#### 特殊的插入和删除：前推/后推，前弹出/后弹出

假设被推入和弹出的元素为`e`并且切片`s`拥有至少一个元素。

```go
// 前弹出（pop front，又称shift）
s, e = s[1:], s[0]
// 后弹出（pop back）
s, e = s[:len(s)-1], s[len(s)-1]
// 前推（push front）
s = append([]T{e}, s...)
// 后推（push back）
s = append(s, e)
```

请注意：使用`append`函数来插入元素常常是比较低效的，因为插入点后的所有元素都要向后挪，并且当空余容量不足时还需要开辟一个更大的内存空间来容纳插入完成后所有的元素。 对于元素个数不多的切片来说，这些可能并不是严重的问题；但是在元素个数很多的切片上进行如上的插入操作常常是耗时的。所以如果元素个数很多，最好使用链表来实现元素插入操作。

#### 关于上面各种切片操控的例子

在实践中，需求是各种各样的。对于某些特定的情形，上面的例子中的代码实现可能并非是最优化的，甚至是不满足要求的。 所以，请在实践中根据具体情况来实现代码。或许，这就是Go没有支持更多的内置切片操作的原因。

### 用映射来模拟集合（set）

Go不支持内置集合（**set**）类型。但是，集合类型可以用轻松使用映射类型来模拟。 在实践中，我们常常使用映射类型`map[K]struct{}`来模拟一个元素类型为`K`的集合类型。 类型`struct{}`的尺寸为零，所以此映射类型的值中的元素不消耗内存。

### 上述各种容器操作内部都未同步

请注意，上述所有各种容器操作的内部实现都未进行同步。如果不使用今后将要介绍的各种并发同步技术，在没有协程修改一个容器值和它的元素的时候，多个协程并发读取此容器值和它的元素是安全的。但是并发修改同一个容器值则是不安全的。 不使用并发同步技术而并发修改同一个容器值将会造成数据竞争。请阅读以后的[并发同步概述](https://gfw.go101.org/article/concurrent-synchronization-overview.html)一文以了解Go支持的各种并发同步技术。