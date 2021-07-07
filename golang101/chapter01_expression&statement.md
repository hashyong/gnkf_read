# 表达式、语句和简单语句

> 表达式：表示值
>
> 语句：表示操作
>
>  另外，根据场合不同，某些语句也可以被视为表达式。

Go中，某些语句被称为简单语句。Go中各种流程控制语句的某些部分可能会被要求必须为简单语句或者表达式。



### 一些表达式的例子

> 单值表达式/多值表达式。Go中大多都是单值表达式

以后（不包括本文），如果没有特殊说明，当表达式这个词被提及的时候，它表示一个单值表达式。

+ 字面量、变量和有名常量等均属于单值表达式。它们可称为基本表达式。

+ 运算符操作（不包括赋值部分）也属于单值表达式。

+ 如果一个函数至少返回一个值，则它的调用属于表达式。 特别的，如果此函数返回两个或两个以上的值，则对它的调用称为多值表达式。 **不返回任何结果的函数的调用不属于表达式。**

+ 上述对函数的解释同样适用于方法。

事实上，以后我们将会了解到自定义函数（包括方法）本身都属于**函数类型**的值，所以它们都是单值表达式。

+ 通道的接收数据操作（不包括赋值部分）也属于表达式。

+ Go中的一些表达式，包括刚提及的通道的接收数据操作，可能会表示可变数量的值。 根据不同的场景，这样的表达式可能呈现为单值表达式，也可能呈现为多值表达式。 我们将在以后的文章中了解到这样的表达式。



### 简单语句类型列表

> Go中有六种简单语句类型：

1.  变量短声明语句。

2.  纯赋值语句，包括`x op= y`这种运算形式。

3.  有返回结果的函数或方法调用，以及通道的接收数据操作。(也是表达式)

4.  通道的发送数据操作。

   ```go
   var a int
   ch := make(chan int)
   a <- ch
   ```

5.  空语句。

6.  自增（`x++`）和自减（`x--`）语句。

   ```go
   foo := bar++ + 10 // 非法
   ```

**注意：**和C/C++不一样，在Go中，自增和自减语句不能被当作表达式使用（即不能赋值给变量）。

简单语句这个概念在Go中比较重要，所以请牢记这六种简单语句类型。

### 一些非简单语句

> 下面是一个非简单语句的不完整列表：

- 标准变量声明语句。
- （有名）常量声明语句。
- 类型声明语句。
- （代码）包引入语句。
- 显式代码块。一个显式代码块起始于一个左大括号`{`，终止于一个右大括号`}`。 一个显式代码块中可以包含若干子语句。
- 函数声明。 一个函数声明中可以包含若干子语句。
- 流程控制跳转语句。详见下一章。
- 函数返回（`return`）语句。
- 延迟函数调用和协程创建语句。

### 一些表达式和语句的例子

```go
// 一些非简单语句：
import "time"
var a = 123
const B = "Go"
type Choice bool
func f() int {
	for a < 10 {
		break
	}

	// 这是一个显式代码块。
	{
		// ...
	}
	return 567
}

// 一些简单语句的例子：
c := make(chan bool) // 通道将在以后讲解
a = 789
a += 5
a = f() // 这是一个纯赋值语句
a++
a--
c <- true // 一个通道发送操作
z := <-c  // 一个使用通道接收操作
          // 做为源值的变量短声明语句

// 一些表达式的例子：
123
true
B
B + " language"
a - 789
a > 0 // 一个类型不确定布尔值
f     // 一个类型为“func ()”的表达式

// 下面这些即可以被视为简单语句，也可以被视为表达式。
f() // 函数调用
<-c // 通道接收操作
```

# 基本流程控制语法

Go中的流程控制语句和其它很多流行语言很类似，但是也有不少区别。 本篇文章将列出所有这些相似点和不同点。

### Go中的流程控制语句简单介绍

> #### 三种基本的流程控制代码块：

- `if-else`条件分支代码块；
- `for`循环代码块；
- `switch-case`多条件分支代码块。

> #### 三种和特定种类的类型相关的流程控制代码块：

- [容器类型](https://gfw.go101.org/article/container.html#iteration)相关的`for-range`循环代码块。

- [接口类型](https://gfw.go101.org/article/interface.html#type-switch)相关的`type-switch`多条件分支代码块。

  ```go
  // 天机阁trace染色
  ```

- [通道类型](https://gfw.go101.org/article/channel.html#select)相关的`select-case`多分支代码块。

> #### 跳转语句

+ 和很多其它流行语言一样，Go也支持`break`、`continue`和`goto`等跳转语句。 另外，Go还支持一个特有的`fallthrough`跳转语句。

> #### 流程控制语句分类

+ Go所支持的六种流程控制代码块中，除了`if-else`条件分支代码块，其它五种称为**可跳出代码块**。 我们可以在一个可跳出代码块中使用`break`语句以跳出此代码块。

+ 我们可以在`for`和`for-range`两种**循环代码块**中使用`continue`语句提前结束一个循环步。 除了这两种循环代码块，其它四种代码块称为分支代码块。

**注意:**  上面所提及的每种流程控制的一个分支都属于一条语句。这样的语句常常会包含很多子语句。

上面所提及的流程控制语句都属于狭义上的流程控制语句。 下一篇文章中将要介绍的[协程、延迟函数调用、以及恐慌和恢复](https://gfw.go101.org/article/control-flows-more.html)，以及今后要介绍的[并发同步技术](https://gfw.go101.org/article/concurrent-synchronization-overview.html)属于广义上的流程控制语句。

本文余下的部分将只解释三种基本的流程控制语句和各种代码跳转语句。其它上面提及的语句将在后面其它文章中逐渐介绍。



### `if-else`条件分支控制代码块

一个`if-else`条件分支控制代码块的完整形式如下：

```go
if InitSimpleStatement; Condition {
	// do something
} else {
	// do something
}
```

`if`和`else`是两个关键字。 和很多其它编程语言一样，**`else`分支是可选的**。

在一个`if-else`条件分支控制代码块中，

- **`InitSimpleStatement`部分是可选的**，如果它没被省略掉，则它必须为一条[简单语句](https://gfw.go101.org/article/expressions-and-statements.html#simple-statements)。 如果它被省略掉，它可以被视为一条空语句（简单语句的一种）。 在实际编程中，`InitSimpleStatement`常常为一条变量短声明语句。
- **`Condition`必须为一个结果为布尔值的[表达式](https://gfw.go101.org/article/expressions-and-statements.html#expressions)（**它被称为条件表达式）。 `Condition`部分可以用一对小括号括起来，但大多数情况下不需要。

注意，我们不能用一对小括号将`InitSimpleStatement`和`Condition`两部分括在一起。

在执行一个`if-else`条件分支控制代码块中，如果`InitSimpleStatement`这条语句没有被省略，则此条语句将被率先执行。 如果`InitSimpleStatement`被省略掉，则其后跟随的分号`;`也可一块儿被省略。

每个`if-else`流程控制包含一个隐式代码块，一个`if`分支显式代码块和一个可选的`else`分支代码块。 这两个分支代码块内嵌在这个隐式代码块中。 在程序运行中，如果`Condition`条件表达式的估值结果为`true`，则`if`分支式代码块将被执行；否则，`else`分支代码块将被执行。

一个例子：

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	if n := rand.Int(); n%2 == 0 {
		fmt.Println(n, "是一个偶数。")
	} else {
		fmt.Println(n, "是一个奇数。")
	}

	n := rand.Int() % 2 // 此n不是上面声明的n
	if n % 2 == 0 {
		fmt.Println("一个偶数。")
	}

	if ; n % 2 != 0 {
		fmt.Println("一个奇数。")
	}
}
```



如果`InitSimpleStatement`语句是一个变量短声明语句，则在此语句中声明的变量被声明在外层的隐式代码块中，**因此需要注意其作用域**。

可选的`else`分支代码块一般情况下必须为显式的，但是如果此分支为另外一个`if-else`块，则此分支代码块可以是隐式的。

另一个例子：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	if h := time.Now().Hour(); h < 12 {
		fmt.Println("现在为上午。")
	} else if h > 19 {
		fmt.Println("现在为晚上。")
	} else {
		fmt.Println("现在为下午。")
		// 左h是一个新声明的变量，右h已经在上面声明了。
		h := h
		// 刚声明的h遮掩了上面声明的h。
		_ = h
	}

	// 上面声明的两个h在此处都不可见。
}
```





### `for`循环代码块

`for`循环代码块的完整形式如下：

```go
for InitSimpleStatement; Condition; PostSimpleStatement {
	// do something
}
```

其中`for`是一个关键字。

在一个`for`循环代码块中，

- `InitSimpleStatement`（初始化语句）和`PostSimpleStatement`（步尾语句）两个部分必须均为简单语句，并且`PostSimpleStatement`不能为一个变量短声明语句。
- `Condition`必须为一个结果为布尔值的表达式（它被称为条件表达式）。

> for的使用注意事项：

+ 所有这三个刚提到的部分都是可选的。和很多其它流行语言不同，**在Go中上述三部分不能用小括号括在一起。**

+ 每个`for`流程控制包括至少两个子代码块。 其中一个是隐式的，另一个是显式的（花括号起始和终止的部分，又称循环体）。 此显式代码块内嵌在隐式代码块之中。

在一个`for`循环流程控制中，初始化语句（`InitSimpleStatement`）将被率先执行，并且只会被执行一次。

在每个循环步的开始，`Condition`条件表达式将被估值。如果估值结果为`false`，则循环立即结束；否则循环体（即显式代码块）将被执行。

在每个循环步的结尾，步尾语句（`PostSimpleStatement`）将被执行。

下面是一个使用`for`循环流程控制的例子。此程序将逐行打印出`0`到`9`十个数字。

```go
for i := 0; i < 10; i++ {
	fmt.Println(i)
}
```



在一个`for`循环流程控制中，如果`InitSimpleStatement`和`PostSimpleStatement`两部分同时被省略（可将它们视为空语句），则和它们相邻的两个分号也可被省略。 这样的形式被称为只有条件表达式的`for`循环。只有条件表达式的`for`循环和很多其它语言中的`while`循环类似。

```go
var i = 0
for ; i < 10; {
	fmt.Println(i)
	i++
}
for i < 20 {
	fmt.Println(i)
	i++
}
```



在一个`for`循环流程控制中，如果条件表达式部分被省略，则编译器视其为`true`。

```go
for i := 0; ; i++ { // 等价于：for i := 0; true; i++ {
	if i >= 10 {
		break
	}
	fmt.Println(i)
}

// 下面这几个循环是等价的。
for ; true; {
}
for true {
}
for ; ; {
}
for {
}
```



在一个`for`循环流程控制中，如果初始化语句`InitSimpleStatement`是一个变量短声明语句，则在此语句中声明的变量被声明在外层的隐式代码块中。 我们可以在内嵌的循环体（显式代码块）中声明同名变量来遮挡在`InitSimpleStatement`中声明的变量。 比如下面的代码打印出`012`，而不是`0`。

```go
for i := 0; i < 3; i++ {
	fmt.Print(i)
	i := i // 这里声明的变量i遮挡了上面声明的i。
	       // 右边的i为上面声明的循环变量i。
	i = 10 // 新声明的i被更改了。
	_ = i
}
```



一条`break`语句可以用来提前跳出包含此`break`语句的最内层`for`循环。 下面这段代码同样逐行打印出`0`到`9`十个数字。

```go
i := 0
for {
	if i >= 10 {
		break
	}
	fmt.Println(i)
	i++
}
```



一条`continue`语句可以被用来提前结束包含此`continue`语句的最内层`for`循环的当前循环步（步尾语句仍将得到执行）。 比如下面这段代码将打印出`13579`。

```go
for i := 0; i < 10; i++ {
	if i % 2 == 0 {
		continue
	}
	fmt.Print(i)
}
```



### `switch-case`流程控制代码块

`switch-case`流程控制代码块是另外一种多分支代码块。

一个`switch-case`流程控制代码块的完整形式为：

```go
switch InitSimpleStatement; CompareOperand0 {
case CompareOperandList1:
	// do something
case CompareOperandList2:
	// do something
...
case CompareOperandListN:
	// do something
default:
	// do something
}
```

其中`switch`、`case`和`default`是三个关键字。

在一个`switch-case`流程控制代码块中，

- `InitSimpleStatement`部分必须为一条简单语句，它是可选的。
- `CompareOperand0`部分必须为一个表达式（如果它没被省略的话，见下）。 此表达式的估值结果总是被视为一个类型确定值。如果它是一个类型不确定值，则它被视为类型为它的默认类型的类型确定值。 因为这个原因，此表达式不能为类型不确定的`nil`值。 `CompareOperand0`常被称为switch表达式。
- 每个`CompareOperandListX`部分（`X`表示`1`到`N`）必须为一个用（英文）逗号分隔开来的表达式列表。 其中每个表达式都必须能和`CompareOperand0`表达式进行比较。 每个这样的表达式常被称为case表达式。 如果其中case表达式是一个类型不确定值，则它必须能够自动隐式转化为对应的switch表达式的类型，否则编译将失败。

每个`case CompareOperandListX:`部分和`default:`之后形成了一个隐式代码块。 每个这样的隐式代码块和它对应的`case CompareOperandListX:`或者`default:`形成了一个分支。 每个分支都是可选的。

每个`switch-case`流程控制代码块中最多只能有一个`default`分支（默认分支）。

除了刚提到的分支代码块，每个`switch-case`流程控制至少包括其它两个代码块。 其中一个是隐式的，另一个是显式的。此显式的代码块内嵌在隐式的代码块之中。 所有的分支代码块都内嵌在此显式代码块之中（因此也间接内嵌在刚提及的隐式代码块中）。

`switch-case`代码块属于可跳出流程控制。 `break`可以使用在一个`switch-case`流程控制的任何分支代码块之中以提前跳出此`switch-case`流程控制。

当一个`switch-case`流程控制被执行到的时候，其中的简单语句`InitSimpleStatement`将率先被执行。 随后switch表达式`CompareOperand0`将被估值（仅一次）。上面已经提到，此估值结果一定为一个类型确定值。 然后此结果值将从上到下从左到右和各个`CompareOperandListX`表达式列表中的各个case表达式逐个依次比较（使用`==`运算符）。 一旦发现某个表达式和`CompareOperand0`相等，比较过程停止并且此表达式对应的分支代码块将得到执行。 如果没有任何一个表达式和`CompareOperand0`相等，则`default`默认分支将得到执行（如果此分支存在的话）。

一个`switch-case`流程控制的例子：

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	switch n := rand.Intn(100); n%9 {
	case 0:
		fmt.Println(n, "is a multiple of 9.")

		// 和很多其它语言不一样，程序不会自动从一个
		// 分支代码块跳到下一个分支代码块去执行。
		// 所以，这里不需要一个break语句。
	case 1, 2, 3:
		fmt.Println(n, "mod 9 is 1, 2 or 3.")
		break // 这里的break语句可有可无的，效果
		      // 是一样的。执行不会跳到下一个分支。
	case 4, 5, 6:
		fmt.Println(n, "mod 9 is 4, 5 or 6.")
	// case 6, 7, 8:
		// 上一行可能编译不过，因为6和上一个case中的
		// 6重复了。是否能编译通过取决于具体编译器实现。
	default:
		fmt.Println(n, "mod 9 is 7 or 8.")
	}
}
```

在上例中，`rand.Intn`函数将返回一个从`0`到所传实参之间类型为`int`的随机数。

注意，编译器可能会不允许一个`switch-case`流程控制中有任何两个case表达式可以在编译时刻确定相等。 比如，当前的官方标准编译器（1.16版本）认为上例中的`case 6, 7, 8`一行是不合法的（如果此行未被注释掉）。但是其它编译器未必这么认为。 事实上，当前的官方标准编译器[允许重复的布尔case表达式在同一个`switch-case`流程控制中出现](https://github.com/golang/go/issues/28357)， 而gccgo（v8.2）允许重复的布尔和字符串类型的case表达式在同一个`switch-case`流程控制中出现。

上面的例子中的前两个`case`分支中的注释已经解释了，和很多其它语言不一样，每个分支代码块的结尾不需要一条`break`语句就可以自动跳出当前的`switch-case`流程控制。 那么如何让执行从一个`case`分支代码块的结尾跳入下一个分支代码块？Go提供了一个`fallthrough`关键字来完成这个任务。 比如，在下面的例子中，所有的分支代码块都将得到执行（从上到下）。

```go
rand.Seed(time.Now().UnixNano())
switch n := rand.Intn(100) % 5; n {
case 0, 1, 2, 3, 4:
	fmt.Println("n =", n)
	fallthrough // 跳到下个代码块
case 5, 6, 7, 8:
	// 一个新声明的n，它只在当前分支代码快内可见。
	n := 99
	fmt.Println("n =", n) // 99
	fallthrough
default:
	// 下一行中的n和第一个分支中的n是同一个变量。
	// 它们均为switch表达式"n"。
	fmt.Println("n =", n)
}
```



请注意：

- 一条`fallthrough`语句必须为一个分支代码块中的最后一条语句。
- 一条`fallthrough`语句不能出现在一个`switch-case`流程控制中的最后一个分支代码块中。

比如，下面代码的几个`fallthrough`使用是不合法的。

```go
switch n := rand.Intn(100) % 5; n {
case 0, 1, 2, 3, 4:
	fmt.Println("n =", n)
	// 此整个if代码块为当前分支中的最后一条语句
	if true {
		fallthrough // error: 不是当前分支中的最后一条语句
	}
case 5, 6, 7, 8:
	n := 99
	fallthrough // error: 不是当前分支中的最后一条语句
	_ = n
default:
	fmt.Println(n)
	fallthrough // error: 不能出现在最后一个分支中
}
```



一个`switch-case`流程控制中的`InitSimpleStatement`语句和`CompareOperand0`表达式都是可选的。 如果`CompareOperand0`表达式被省略，则它被认为类型为`bool`类型的`true`值。 如果`InitSimpleStatement`语句被省略，其后的分号也可一并被省略。

上面已经提到了一个`switch-case`流程控制中的所有分支都可以被省略，所以下面的所有流程控制代码块都是合法的，它们都可以被视为空操作。

```go
switch n := 5; n {
}

switch 5 {
}

switch _ = 5; {
}

switch {
}
```



上例中的后两个`switch-case`流程控制中的`CompareOperand0`表达式都为`bool`类型的`true`值。 同理，下例中的代码将打印出`hello`。

```go
switch { // <=> switch true {
case true: fmt.Println("hello")
default: fmt.Println("bye")
}
```



Go中另外一个和其它语言的显著不同点是`default`分支不必一定为最后一个分支。 比如，下面的三个`switch-case`流程控制代码块是相互等价的。

```go
switch n := rand.Intn(3); n {
case 0: fmt.Println("n == 0")
case 1: fmt.Println("n == 1")
default: fmt.Println("n == 2")
}

switch n := rand.Intn(3); n {
default: fmt.Println("n == 2")
case 0: fmt.Println("n == 0")
case 1: fmt.Println("n == 1")
}

switch n := rand.Intn(3); n {
case 0: fmt.Println("n == 0")
default: fmt.Println("n == 2")
case 1: fmt.Println("n == 1")
}
```



### `goto`跳转语句和跳转标签声明

和很多其它语言一样，Go也支持`goto`跳转语句。 在一个`goto`跳转语句中，`goto`关键字后必须跟随一个表明跳转到何处的跳转标签。 我们使用`LabelName:`这样的形式来声明一个名为`LabelName`的跳转标签，其中`LabelName`必须为一个标识符。 一个不为空标识符的跳转标签声明后必须被使用至少一次。

一条跳转标签声明之后必须立即跟随一条语句。 如果此声明的跳转标签使用在一条`goto`语句中，则当此条`goto`语句被执行的时候，执行将跳转到此跳转标签声明后跟随的语句。

一个跳转标签必须声明在一个函数体内，此跳转标签的使用可以在此跳转标签的声明之后或者之前，但是此跳转标签的使用不能出现在此跳转标签声明所处的最内层代码块之外。

下面这个例子使用跳转标签声明和`goto`跳转语句来实现了一个循环：

```go
package main

import "fmt"

func main() {
	i := 0

Next: // 跳转标签声明
	fmt.Println(i)
	i++
	if i < 5 {
		goto Next // 跳转
	}
}
```



上面刚提到了一个跳转标签的使用不能出现在此跳转标签声明所处的最内层代码块之外，所以下面的代码片段中的跳转标签使用都是不合法的。

```go
package main

func main() {
goto Label1 // error
	{
		Label1:
		goto Label2 // error
	}
	{
		Label2:
	}
}
```



另外要注意的一点是，如果一个跳转标签声明在某个变量的作用域内，则此跳转标签的使用不能出现在此变量的声明之前。 关于变量的作用域，请阅读后面的文章[代码块和作用域](https://gfw.go101.org/article/blocks-and-scopes.html)

下面这个程序编译不通过：

```go
package main

import "fmt"

func main() {
	i := 0
Next:
	if i >= 5 {
		// error: goto Exit jumps over declaration of k
		goto Exit
	}

	k := i + i
	fmt.Println(k)
	i++
	goto Next
Exit: // 此标签声明在k的作用域内，但
      // 它的使用在k的作用域之外。
}
```

刚提到的这条规则[可能会在今后放宽](https://github.com/golang/go/issues/26058)。 目前，有两种途径可以对上面的程序略加修改以使之编译通过。

第一种途径是缩小变量`k`的作用域：

```go
func main() {
	i := 0
Next:
	if i >= 5 {
		goto Exit
	}
	// 创建一个显式代码块以缩小k的作用域。
	{
		k := i + i
		fmt.Println(k)
	}
	i++
	goto Next
Exit:
}
```

第二种途径是放大变量`k`的作用域：

```go
func main() {
	var k int // 将变量k的声明移到此处。
	i := 0
Next:
	if i >= 5 {
		goto Exit
	}

	k = i + i
	fmt.Println(k)
	i++
	goto Next
Exit:
}
```



### 包含跳转标签的`break`和`continue`语句

一个`goto`语句必须包含一个跳转标签名。 一个`break`或者`continue`语句也可以包含一个跳转标签名，但此跳转标签名是可选的。 包含跳转标签名的`break`语句一般用于跳出外层的嵌套可跳出流程控制代码块。 包含跳转标签名的`continue`语句一般用于提前结束外层的嵌套循环流程控制代码块的当前循环步。

如果一条`break`语句中包含一个跳转标签名，则此跳转标签必须刚好声明在一个包含此`break`语句的可跳出流程控制代码块之前。 我们可以把此跳转标签名看作是其后紧跟随的可跳出流程控制代码块的名称。 此`break`语句将立即结束此可跳出流程控制代码块的执行。

如果一条`continue`语句中包含一个跳转标签名，则此跳转标签必须刚好声明在一个包含此`continue`语句的循环流程控制代码块之前。 我们可以把此跳转标签名看作是其后紧跟随的循环流程控制代码块的名称。 此`continue`语句将提前结束此循环流程控制代码块的当前步的执行。

下面是一个使用了包含跳转标签名的`break`和`continue`语句的例子。

```go
package main

import "fmt"

func FindSmallestPrimeLargerThan(n int) int {
Outer:
	for n++; ; n++{
		for i := 2; ; i++ {
			switch {
			case i * i > n:
				break Outer
			case n % i == 0:
				continue Outer
			}
		}
	}
	return n
}

func main() {
	for i := 90; i < 100; i++ {
		n := FindSmallestPrimeLargerThan(i)
		fmt.Print("最小的比", i, "大的素数为", n)
		fmt.Println()
	}
}
```