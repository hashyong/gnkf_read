# 《GO语言101》

## 第一章 Go编程入门

###  关键字和表标识符

> #### 关键字 

关键字是一些特殊的用来帮助编译器理解和解析源代码的单词。Go中共有25个关键字：

```
break     default      func    interface  select
case      defer        go      map        struct
chan      else         goto    package    switch
const     fallthrough  if      range      type
continue  for          import  return     var
```

> **不常用关键字介绍** 
> fallthrough: switch-case语句中默认执行break， 本关键字屏蔽break，继续向下执行
>
> ```go
> 	var flag bool
> 	switch flag {
> 	case false:
> 		fmt.Print("hello,")
> 		fallthrough
> 	case true:
> 		fmt.Println("world")
> 	}
> // output：hello,world
> ```
>
> goto: 无条件跳转
>
> ```go
> 	goto label
> 	fmt.Print("go")
> 	label: fmt.Print("hello,world")
> // output：hello,world
> ```
>
> chan：定义channel
>
> select : 监听channel事件就绪，类别linux i/o select
>
> ```go
> select {
> case v1 := <-c1:
>     fmt.Printf("received %v from c1\n", v1)
> case v2 := <-c2:
>     fmt.Printf("received %v from c2\n", v1)
> case c3 <- 23:
>     fmt.Printf("sent %v to c3\n", 23)
> default:
>     fmt.Printf("no one was ready to communicate\n")
> }
> // 1.除 default 外，如果只有一个 case 语句评估通过，那么就执行这个case里的语句；
> // 2.除 default 外，如果有多个 case 语句评估通过，那么通过伪随机的方式随机选一个；
> // 3.如果 default 外的 case 语句都没有通过评估，那么执行 default 里的语句；
> // 4.如果没有 default，那么 代码块会被阻塞，指导有一个 case 通过评估；否则一直阻塞
> ```
>
> defer：用于函数延迟执行
>
> ```go
> func createPost(db *gorm.DB) error {
>     tx := db.Begin()
>     defer tx.Rollback()
> 
>     if err := tx.Create(&Post{Author: "Draveness"}).Error; err != nil {
>         return err
>     }
> 
>     return tx.Commit().Error
> }
> // 1.多个defer关键字修饰的函数执行以栈顺序执行
> // 2.defer语句执行时机为return语句修改返回值后，因此defer中可以观测到函数返回值
> // 3.defer语句常用于收尾工作（处理异常、释放资源、关闭连接等），比如上面例子中的回滚
> ```

> #### **标识符**
>
> 标识符：以Unicode字母或者_开头并且完全由Unicode字母和Unicode数字组成的单词
> **导出标识符：**由Unicode大写字母开头的标识符
>
> ```go
> Player_9
> DoSomething
> VERSION
> Ĝo
> Π
> ```
>
> **非导出标识符:** 非Unicode大写字母开头的标识符
>
> ```go
> _status
> memStat
> book
> π
> 一个类型
> 변수
> エラー
> ```
>
> **_ ：**空标识符
>
> ```go
> // 1. 引入某包只执行包中的init函数，但本包没有直接引用该包任何变量或函数，使用import _避免编译错误；
> import _ "net/http/pprof"
> // 2. 函数有多返回值，忽略其中某些返回值。类似c++11 std::ignore在std::tie中的运用；
> 
> 3. -编译期-检查，比如某类型有没有实现某接口的检查；
> _,ok : val.(int)
> 4. 在main之前执行某段代码
> 
> _  :=  getlength(r)
> ```

### 基本类型和字面量

> ####  **基本类型**
>
> 内置基本类型
>
> | 类型            | 零值   | 范围                  |
> | --------------- | ------ | --------------------- |
> | bool            | false  | false,true            |
> | int8            | 0      | -2^7~2^7-1            |
> | uint8/byte      | 0      | 0~2^8-1               |
> | int16           | 0      |                       |
> | uint16          | 0      |                       |
> | int32/rune      | 0      |                       |
> | uint32          | 0      |                       |
> | int64           | 0      |                       |
> | uin64           | 0      |                       |
> | int             | 0      |                       |
> | uint            | 0      |                       |
> | uintptr         | 0      |                       |
> | complex64       | 0      | 实部和虚部都是float32 |
> | complex128      | 0      | 实部和虚部都是float64 |
> | string          | ""或`` |                       |
> | float32/float64 | 0      | IEEE754标准表示范围   |
>
> 注：零值为该类型声明时的默认值
>
> **自定义类型声明**
>
> ```go
> // 一些类型定义声明
> type status bool     // status和bool是两个不同的类型
> type MyString string // MyString和string是两个不同的类型
> type Id uint64       // Id和uint64是两个不同的类型
> type real float32    // real和float32是两个不同的类型
> 
> // 一些类型别名声明
> type boolean = bool // boolean和bool表示同一个类型
> type Text = string  // Text和string表示同一个类型
> type U8 = uint8     // U8、uint8和 byte表示同一个类型
> type char = rune    // char、rune和int32表示同一个类型
> ```
>
> **整型字面量**
>
> ```go
> 0xF // 十六进制表示（必须使用0x或者0X开头）
> 0XF
> 
> 017 // 八进制表示（必须使用0、0o或者0O开头）
> 0o17
> 0O17
> 
> 0b1111 // 二进制表示（必须使用0b或者0B开头）
> 0B1111
> 
> 15  // 十进制表示（必须不能用0开头）
> ```
>
> **浮点数类型字面量**  
>
> | 进制     | 表示形式                                |
> | -------- | --------------------------------------- |
> | 十进制   | xen或xEn(x\*10^n)  xe-n或xE-n(x\*10^-n) |
> | 十六进制 | ypn或yPn (y\*2^n)  yp-n或yP-n (y\*2^-n) |
>
> ```go
> // 十进制浮点型字面量
> 1.23
> 01.23 // == 1.23
> .23
> 1.
> // 一个e或者E随后的数值是指数值（底数为10）。
> // 指数值必须为一个可以带符号的十进制整数字面量。
> 1.23e2  // == 123.0
> 123E2   // == 12300.0
> 123.E+2 // == 12300.0
> 1e-1    // == 0.1
> .1e0    // == 0.1
> 0010e-2 // == 0.1
> 0e+5    // == 0.0
> 
> // 十六进制浮点型字面量
> 0x1p-2     // == 1.0/4 = 0.25
> 0x2.p10    // == 2.0 * 1024 == 2048.0
> 0x1.Fp+0   // == 1+15.0/16 == 1.9375
> 0X.8p1     // == 8.0/16 * 2 == 1.0
> 0X1FFFP-16 // == 0.1249847412109375
> ```
>
> 注意 ```0x15e-2 // == 0x15e - 2 (整数相减表达式)```不是浮点型十六进制字面量
>
> 复数的实部和虚部都是浮点型，这里不再做介绍
>
> 
>
> **数值字面表示中使用下划线分段**
>
> 从Go 1.13开始，下划线`_`可以出现在整数、浮点数和虚部数字面量中，以用做分段符以增强可读性。 但是要注意，在一个数值字面表示中，一个下划线`_`不能出现在此字面表示的首尾，并且其两侧的字符必须为（相应进制的）数字字符或者进制表示头。
>
> ```go
> // 合法的使用下划线的例子
> 6_9          // == 69
> 0_33_77_22   // == 0337722
> 0x_Bad_Face  // == 0xBadFace
> 0X_1F_FFP-16 // == 0X1FFFP-16
> 0b1011_0111 + 0xA_B.Fp2i
> 
> // 非法的使用下划线的例子
> _69        // 下划线不能出现在首尾
> 69_        // 下划线不能出现在首尾
> 6__9       // 下划线不能相连
> 0_xBadFace // x不是一个合法的八进制数字
> 1_.5       // .不是一个合法的十进制数字
> 1._5       // .不是一个合法的十进制数字
> ```
>
> **rune值的字面量形式**
>
> 方式1：整型字面量
>
> 方式2：Unicode码点。一个rune字面量由若干包在一对单引号中的字符组成。 包在单引号中的字符序列表示一个Unicode码点值。rune的Unicode码点字面量表示存在几种不同形式，如下：
>
> ```go
> 'a' // 一个英文字符
> '\141'   // 141是97的八进制表示    \之后必须跟随三个八进制数字字符（0-7）表示一个byte值
> '\x61'   // 61是97的十六进制表示   \x之后必须跟随两个十六进制数字字符（0-9，a-f和A-F）表示一个byte值
> '\u0061' //  \u之后必须跟随四个十六进制数字字符表示一个rune值（此rune值的高四位都为0）
> '\U00000061' //\U之后必须跟随八个十六进制数字字符表示一个rune值
> ```
>
>  这些八进制和十六进制的数字字符序列表示的整数必须是一个合法的Unicode码点值，否则编译将失败。
>
> 如果一个rune字面量中被单引号包起来的部分含有两个字符， 并且第一个字符是`\`，第二个字符不是`x`、 `u`和`U`，那么这两个字符将被转义为一个特殊字符。 目前支持的转义组合为：
>
> ```
> \a   (rune值：0x07) 铃声字符
> \b   (rune值：0x08) 退格字符（backspace）
> \f   (rune值：0x0C) 换页符（form feed）
> \n   (rune值：0x0A) 换行符（line feed or newline）
> \r   (rune值：0x0D) 回车符（carriage return）
> \t   (rune值：0x09) 水平制表符（horizontal tab）
> \v   (rune值：0x0b) 竖直制表符（vertical tab）
> \\   (rune值：0x5c) 一个反斜杠（backslash）
> \'   (rune值：0x27) 一个单引号（single quote）
> ```
>
> **字符串值的字面量形式**
>
> 在Go中，字符串值是UTF-8编码的， 甚至所有的Go源代码都必须是UTF-8编码的。
>
> Go字符串的字面量形式有两种。 一种是解释型字面表示（interpreted string literal，双引号风格）。 另一种是直白字面表示（raw string literal，反引号风格）。 下面的两个字符串表示形式是等价的：
>
> ```go
> // 解释形式
> "Hello\nworld!\n\"你好世界\""
> 
> // 直白形式
> `Hello
> world!
> "你好世界"`
> ```
>
> 在上面的解释形式（双引号风格）的字符串字面量中，每个`\n`将被转义为一个换行符，每个`\"`将被转义为一个双引号字符。 双引号风格的字符串字面量中支持的转义字符和rune字面量基本一致，除了一个例外：双引号风格的字符串字面量中支持`\"`转义，但不支持`\'`转义；而rune字面量则刚好相反。
>
> 以`\`、`\x`、`\u`和`\U`开头的rune字面量（不包括两个单引号）也可以出现在双引号风格的字符串字面量中。比如：
>
> ```go
> // 这几个字符串字面量是等价的。
> "\141\142\143"
> "\x61\x62\x63"
> "\x61b\x63"
> "abc"
> 
> // 这几个字符串字面量是等价的。
> "\u4f17\xe4\xba\xba"
>  			// “众”的Unicode值为4f17，它的UTF-8
>       // 编码为三个字节：0xe4 0xbc 0x97。
> "\xe4\xbc\x97\u4eba"
>       // “人”的Unicode值为4eba，它的UTF-8
>       // 编码为三个字节：0xe4 0xba 0xba。
> "\xe4\xbc\x97\xe4\xba\xba"
> "众人"
> ```
>
> **字符串中包含非ASCII码的一个坑**
>
> ```go
> // URIEncode url编码，实现复刻C++版sdk逻辑  正常版本
> func URIEncode(srcURI string) string {
> 	var encodeURI []byte
> 	for i := 0; i < len(srcURI); i++ {
> 		c := srcURI[i]    // 按byte[]的方式遍历string
> 		if (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '-' ||
> 			c == '_' || c == '.' || c == '~' {
> 			encodeURI = append(encodeURI, c)
> 		} else if c <= 0x20 || c >= 0x7F || strings.Contains(ILLEGAL, string(c)) ||
> 			strings.Contains(ReservedQueryParam, string(c)) {
> 			encodeURI = append(encodeURI, '%')
> 			encodeURI = append(encodeURI, "0123456789ABCDEF"[c>>4])
> 			encodeURI = append(encodeURI, "0123456789ABCDEF"[c&0xF])
> 		} else {
> 			encodeURI = append(encodeURI, c)
> 		}
> 	}
> 	return string(encodeURI)
> }
> 
> // URIEncode url编码，实现复刻C++版sdk逻辑  出错版本1
> func URIEncode(srcURI string) string {
> 	var encodeURI []byte
> 	for _,c := range srcURI {    // 按[]rune的方式遍历string
> 		if (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '-' ||
> 			c == '_' || c == '.' || c == '~' {
> 			encodeURI = append(encodeURI, c)
> 		} else if c <= 0x20 || c >= 0x7F || strings.Contains(ILLEGAL, string(c)) ||
> 			strings.Contains(ReservedQueryParam, string(c)) {
> 			encodeURI = append(encodeURI, '%')
> 			encodeURI = append(encodeURI, "0123456789ABCDEF"[c>>4])
> 			encodeURI = append(encodeURI, "0123456789ABCDEF"[c&0xF])
> 		} else {
> 			encodeURI = append(encodeURI, c)
> 		}
> 	}
> 	return string(encodeURI)
> }
> 
> 
> // URIEncode url编码，实现复刻C++版sdk逻辑  出错版本2
> func URIEncode(srcURI string) string {
> 	var encodeURI []byte
> 	for i,_ := range srcURI {    
>     c := srcURI[i] // 用[]rune的下标按[]byte的方式遍历string
> 		if (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c == '-' ||
> 			c == '_' || c == '.' || c == '~' {
> 			encodeURI = append(encodeURI, c)
> 		} else if c <= 0x20 || c >= 0x7F || strings.Contains(ILLEGAL, string(c)) ||
> 			strings.Contains(ReservedQueryParam, string(c)) {
> 			encodeURI = append(encodeURI, '%')
> 			encodeURI = append(encodeURI, "0123456789ABCDEF"[c>>4])
> 			encodeURI = append(encodeURI, "0123456789ABCDEF"[c&0xF])
> 		} else {
> 			encodeURI = append(encodeURI, c)
> 		}
> 	}
> 	return string(encodeURI)
> }
> ```
>
> 所以，在包含非ASCII码或Unicode码点值大于8位的字符时，需要注意用正确的方式遍历
>
> ```go
> // 需要对非ASCII码或Unicode码点值大于8位的字符进行正确操作，截取[]rune
> s := "截取a中文"
> res := []rune(s)
> fmt.Println(string(res[:2]))
> // output: 截取
> 
> // 错误做法, 截取[]byte
> s := "截取中文"
> fmt.Println(s[:2]) //??
> ```

> **基本数值类型字面量的适用范围**

一个数值型的字面量只有在不需要舍入时，才能用来表示一个整数基本类型的值。 比如，`1.0`可以表示任何基本整数类型的值，但`1.01`却不可以。 当一个数值型的字面量用来表示一个非整数基本类型的值时，舍入（或者精度丢失）是允许的。

每种数值类型有一个能够表示的数值范围。 如果一个字面量超出了一个类型能够表示的数值范围（溢出），则在编译时刻，此字面量不能用来表示此类型的值。

下表是一些例子：

|             字面表示             |         此字面表示可以表示哪些类型的值（在编译时刻）         |
| :------------------------------: | :----------------------------------------------------------: |
|              `256`               |         除了int8和uint8类型外的所有的基本数值类型。          |
|              `255`               |             除了int8类型外的所有的基本数值类型。             |
|              `-123`              |          除了无符号整数类型外的所有的基本数值类型。          |
|              `123`               |                     所有的基本数值类型。                     |
|            `123.000`             |                                                              |
|             `1.23e2`             |                                                              |
|              `'a'`               |                                                              |
|             `1.0+0i`             |                                                              |
|              `1.23`              |                所有浮点数和复数基本数值类型。                |
| `0x10000000000000000` (16 zeros) |                                                              |
|             `3.5e38`             | 除了float32和complex64类型外的所有浮点数和复数基本数值类型。 |
|              `1+2i`              |                    所有复数基本数值类型。                    |
|             `2e+308`             |                             无。                             |

注意几个溢出的例子：

- 字面量`0x10000000000000000`需要65个比特才能表示，所以在运行时刻，任何基本整数类型都不能精确表示此字面量。
- 在IEEE-754标准中，最大的可以精确表示的float32类型数值为`3.40282346638528859811704183484516925440e+38`，所以`3.5e38`不能表示任何float32和complex64类型的值。
- 在IEEE-754标准中，最大的可以精确表示的float64类型数值为`1.797693134862315708145274237317043567981e+308`，因此`2e+308`不能表示任何基本数值类型的值。
- 尽管`0x10000000000000000`可以用来表示float32类型的值，但是它不能被任何float32类型的值所精确表示。上面已经提到了，当使用字面量来表示非整数基本数值类型的时候，精度丢失是允许的（但溢出是不允许的）。
