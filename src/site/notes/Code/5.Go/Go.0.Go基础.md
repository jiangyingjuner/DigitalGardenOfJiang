---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.0.Go基础/","title":"Go基础","noteIcon":""}
---


# Go基础

## 函数

**Go 语言的函数参数传递，只有值传递，没有引用传递**

### 可变参数

> [了解golang的可变参数(... parameters)，这一篇就够了 - apocelipes - 博客园 (cnblogs.com)](https://www.cnblogs.com/apocelipes/p/9861315.html)

在声明可变参数函数时，需要在参数列表的最后一个参数类型之前加上省略符号 `…`，这表示该函数会接收任意数量的该类型参数

可变参数的传递可以使用 `…` 运算符以变量 `…` 的形式进行参数传递，这里的变量必须是与可变参数类型相同的 slice，而不能是其他类型(数组也不可以)

```go
numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
sum(numbers...) // 和sum(1, 2, 3, 4, 5, 6, 7, 8, 9. 10)等价
```

**只能对slice类型使用…运算符**

```go
arr := [...]int{1, 2, 3, 4, 5}
sum(arr...) // 编译无法通过,cannot use arr (type [5]int) as type []int in argument to sum
```

`…`运算符是个语法糖，**它把前面的slice直接复制给可变参数，而不是先解包成独立的n个参数再传递**

**不能把独立传参和…运算符混用**

```go
slice := []int{2, 3, 4, 5}
sum(1, slice...) // 无法通过编译
// too many arguments in call to sum
//    have (number, []int...)
//    want (...int)
```

`…` 运算符将不定参数直接替换成了 slice，这样就导致前一个独立给出的参数不再算入可变参数的范围内，使得函数的参数列表从 `(…int)` 变成了 `(int, …int)`，最终使得函数类型不匹配编译失败

正确的做法也很简单，不要混合使用…运算符给可变参数传参即可。

### For

#### range

**`for index,value := range(array)`**，类似于
```Go
len_temp := len(array)
array_temp := array
var index int
value := array[0]
for index_temp = 0; index_temp < len_temp; index_temp++ {
  value_temp = array_temp[index_temp]
  index = index_temp
  value = value_temp
  /*
   * original body
   */              
 }  
```
因此具有如下特点：
1. 在循环中 value 的地址始终不变
2. 循环开始后 index 和 value 的值确定，不再受原始 array 的更改影响
3. For range 遍历的内容是对原内容的一个拷贝，所以不能用来修改原切片中内容

**`for key,value := range map`**
```go
var hiter map_iteration_struct
for mapiterinit(type, range, &hiter); hiter.key != nil; mapiternext(&hiter) {
  index_temp = *hiter.key
  value_temp = *hiter.val
  index = index_temp
  value = value_temp
  /*
   * original body
   */ 
}
```
由于 map 底层实现与 slice 不同，底层使用 hash 表实现，插入数据位置随机，所以遍历过程中新插入的数据不一定会被遍历到

**`for value := range channel`**
```go
for {
  value_temp, ok_temp = <-channel
  if !ok_temp {
    break
  }
  value = value_temp
  /*
   * original body
   */ 
}
```
使用 for-range 遍历 channel 时只能获取一个返回值
如果 channel 中没有数据，可能会阻塞

此外，Go 语言遍历数组或者切片并删除全部元素的逻辑会被优化为直接清除目标数组内存空间中的全部数据

### switch

```go
switch simpleStatement; condition {
    case expression1,expression2:
        statements
    case expression3:
        statements
        fallthrough
    default:
        statements
}
```

Go 语言中匹配到一个 case 条件执行完对应的逻辑之后就会跳出这个 switch 语句，等价于每个 case 处理逻辑之后都加了一个隐式的 break 语句
如果不想要隐式退出的话可以使用 fallthrough 语句来继续下一个 case 的处理逻辑
同时也可以显示调用 break 提前跳出

switch 条件还可以用于类型判断
```go
i := interface{}(123)
switch i.(type) {
case int:
    println("int")
case int64:
    println("int64")
case float64:
    println("float64")
}
```
### make & new

`make` 函数只用于 map，slice 和 channel 初始化，并且不返回指针，因为这三种类型本身即为引用类型
`new` 函数只接受一个类型参数，并且返回一个指向该类型内存地址的指针，且将其中类型初始化为0值，指针初始化为 nil

### Method 方法

>  [值接收者和指针接收者的区别 | Go 程序员面试笔试宝典 (golang.design)](https://golang.design/go-questions/interface/receiver/)

Method 是绑定在某种类型的变量(结构体或基础类型的别名)上的特殊的函数

类型和作用在它上面的方法必须在同一个包里定义

```Go
func (recv recvType) methodName(paramrList) (returnValList) { ... }
```

`recvType` 既可以为值也可以为指针
Method 既可以通过变量指针调用也可以通过变量调用，调用方法相同，都为 `recv.methodName(paramrList)`

|                | 值接收者                                 | 指针接收者                                                                   |
| -------------- | ---------------------------------------- | ---------------------------------------------------------------------------- |
| 值类型调用者   | 方法会使用调用者的一个副本，类似于“传值” | 使用值的引用来调用方法                                                       |
| 指针类型调用者 | 指针被解引用为值，使用该值的副本         | 实际上也是“传值”，方法里的操作会影响到调用者，类似于指针传参，拷贝了一份指针 |

在进行多态调用的时候，必须满足如下规则

|                | 结构体实现接口 | 结构体指针实现接口 |
|:--------------:|:--------------:|:------------------:|
|   结构体变量   |    编译通过    |     编译不通过     |
| 结构体指针变量 |    编译通过    |      编译通过      |

**类型 `T` 只有接受者是 `T` 的方法；而类型 `*T` 拥有接受者是 `T` 和 `*T` 的方法，语法上 `T` 能直接调 `*T` 的方法仅仅是 `Go` 的语法糖**

```go
type animal interface {
	walk()
	bark()
}
type Dog struct {
	kind string
}

func (d Dog) walk() {
	fmt.Printf("A %s is walking", d.kind)
}
func (d *Dog) bark() {
	fmt.Printf("A %s is barking", d.kind)
}
func isAnimal(a animal) {
	fmt.Println("It is an animal")
}

func main() {
	ptrd := &Dog{"dog"}
	d := Dog{"dog"}
	ptrd.walk()          // 编译通过，解引用ptrd调用walk()
	ptrd.bark()          // 编译通过，拷贝一份ptrd调用bark()
	d.walk()             // 编译通过，使用d的一个副本调用walk()
	d.bark()             // 编译通过，使用d的引用来调用bark()
	isAnimal(ptrd)       // 编译通过，自动实现了指针类型的walk()
	// isAnimal(d)       // 编译错误，d没有实现bark(),不是一个animal
}

```

### 匿名函数&闭包

>[!Cite] 参考
> [Golang：“闭包(closure)”到底包了什么？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/92634505)

闭包的定义是由函数及其相关的引用环境组合而成的实体

闭包通常通过调用一个**外部函数**返回其**内部匿名函数**来实现

在闭包**第一次实际运行**时，其保存相关的引用环境**(闭包捕获的变量和常量都是引用传递，不是值传递**)，直到闭包的生命周期结束

**延迟绑定**：闭包内变量的初始值为闭包**第一次实际运行**(而非定义)时变量的最新值，同时闭包内的变量不随原始变量本身的生命周期结束而消失
- 当闭包内变量为指针时，也同时保证了指针指向的值不消失

## 错误

### Error

### Panic

程序执行的异常会触发 panic，**panic 触发后立即执行并仅执行在该 goroutine 中的 defer 函数**，随后崩溃程序，输出包括 panic value 和函数调用的堆栈跟踪信息的日志

### Defer

>  [Golang中的Defer必掌握的7知识点 - Go语言中文网 - Golang中文社区 (studygolang.com)](https://studygolang.com/articles/27408)

> [【GoLang】defer的坑与应用 - 第一节_Gnight_jmup的博客-CSDN博客](https://blog.csdn.net/weixin_44626319/article/details/119581767?spm=1001.2014.3001.5502)

同一局部空间内 defer 以堆栈方式存取，因此**后声明的 defer 先执行**

**Defer 在声明时立刻对调用的参数进行求值**，但具体的函数调用直到周围的函数返回才执行

Return 过程可以被分解为以下三步：
1. 返回值赋值
2. 执行 defer 语句
3. 将结果返回
因此若当前函数声明中命名了返回值变量，可以在 defer 函数中对该返回值进行修改

**循环体中不应使用 defer 调用语句**，一方面是会影响性能，另一方面是可能会发生一些意想不到的结果

在 defer 语句中设置 panic 会覆盖掉函数中的 panic

#### 具体实现

进行 defer 函数调用时生成一个 `_defer` 结构，一个函数中可能有多次 defer 调用，所以会生成多个`_defer` 结构，链式存储构成一个`_defer` 链表，当前 goroutine 的 `_defer` 指向这个链表的头节点

```Go
type _defer struct {
   started bool        // 标志位，标识defer函数是否已经开始执行,默认为false
   heap    bool        // 标记位，标志当前defer结构是否是分配在堆上
   openDefer bool      // 标记位，标识当前defer是否以开放编码的方式实现
   sp        uintptr   // 调用方的sp寄存器指针，即栈指针
   pc        uintptr   // 调用方的程序计数器指针
   fn        func()    // defer注册的延迟执行的函数
   _panic    *_panic   // 标识是否panic时触发，非panic触发时，为nil
   link      *_defer   // defer链表
   fd   unsafe.Pointer // defer调用的相关参数
   varp uintptr        // value of varp for the stack frame
   framepc uintptr
}
```

Defer 的实现有三种实现方式
- 优先使用内联方式
- 当内联不满足，且没有发生内存逃逸的情况下，使用栈分配的方式
- 这两种情况都不符合的情况下使用堆分配

### Recover

`recover()` 函数可以捕获 `panic` 阻止程序中断，**必须在 `defer` 函数中直接调用 `recover` 函数**

当一个函数发生了 panic 之后
- 如果在当前函数中没有 recover，会一直向外层传递直到主函数，最终中止协程
- 如果在过程中遇到了 recover 则被捕获，执行完 recover 后结束函数，返回返回值类型的零值

限制
- recover 只能捕获本协程内的 panic
- 利用 recover 处理 panic 指令，defer 必须在 panic 之前被声明
- 一个 recover 只能捕获当前的最后一个 panic

可以捕获的异常
- **array/slice 下标越界**
- **空指针异常**
- **往已经 close 的 chan 中发送数据**
- **类型断言**
#### 捕获失败

`defer` 中调用 `recover` 函数的包装函数或是在嵌套的 `defer` 函数中调用 `recover` 函数都会导致捕获失败
```go
func MyRecover() interface{} {
    log.Println("trace...")
    return recover()
}

func main() {
    defer func() {
        // 无法捕获异常
        if r := MyRecover(); r != nil {
            fmt.Println(r)
        }
    }()
    panic(1)
}

func main() {
    defer func() {
        defer func() {
            // 无法捕获异常
            if r := recover(); r != nil {
                fmt.Println(r)
            }
        }()
    }()
    panic(1)
}

func main() {
    // 无法捕获异常
    defer recover()
    panic(1)
}


func MyRecover() interface{} {
    return recover()
}

func main() {
    // 可以正常捕获异常
    defer MyRecover()
    panic(1)
}

```

必须要和有异常的栈帧只隔一个栈帧，`recover` 函数才能正常捕获异常
换言之，`recover` 函数捕获的是祖父一级调用函数栈帧的异常(刚好可以跨越一层 `defer` 函数)

此外以下情况 `recover` 也会捕获失败，主要原因是代码直接使用了 `throw`，退出了运行时
  - **内存溢出**：通过 `make` 申请大量内存，其本质是执行了 `mmap` 命令，其内存不够直接内部 `throw`，抛出错误
  - **map 并发读写**：对该特性的解释是 go 的设计希望不要编译好之后在运行时检测，而是要在 `--race` 条件下编译
  - **栈内存耗尽**：在栈的扩张中，会校验新的栈大小是否超过阈值 `1 << 20`
  - **尝试将 nil 函数交给 goroutine 启动**
  - **所有线程都休眠**：死锁
  - **重复解锁互斥锁**

## 泛型

>  [泛型 | 李文周的博客 (liwenzhou.com)](https://www.liwenzhou.com/posts/Go/generics/)

Go 1.18版本增加了对泛型的支持

泛型为 Go 语言添加了三个新的重要特性
- 函数和类型的类型参数
- 将接口类型定义为类型集，包括没有方法的类型
- 类型推断，允许在调用函数时在许多情况下省略类型参数

Go 语言中的函数和类型支持添加类型参数，类型参数列表看起来像普通的参数列表，只不过使用方括号 `[]`
对于类型参数，可以从实参的类型推断出函数的类型实参，但只适用于函数参数中使用的类型参数，而不适用于仅在函数结果中或仅在函数体中使用的类型参数


函数
```go
// 类型约束字面量，通常外层interface{}可省略
func min[T interface{ int | float64 }](a, b T) T {
	if a <= b {
		return a
	}
	return b
}

// 事先定义好的类型约束类型
type Value interface {
	int | float64
}
func min[T Value](a, b T) T {
	if a <= b {
		return a
	}
	return b
}

var a, b, m float64
m = min[float64](a, b) // 显式指定类型实参
m = min(a, b)          // 无需指定类型实参
```

类型
```go
type Slice[T int | string] []T

type Map[K int | string, V float32 | float64] map[K]V

type Tree[T interface{}] struct {
	left, right *Tree[T]
	value       T
}

func (t *Tree[T]) Lookup(x T) *Tree[T] { ... }
```

### 约束类型推断

```go
func Scale[S ~[]E, E constraints.Integer](s S, c E) S {
    r := make(S, len(s))
    for i, v := range s {
        r[i] = v * c
    }
    return r
}

type Point []int32
var p Point{1,2,3}
Scale(p,2)
```

对于 `Scale` 函数，当 S 的类型确定时，编译器通过约束类型推断从类型参数约束推导类型参数，由已知的类型 `Point` 推断出 `E` 的类型 `int32`

## Go 初始化

程序的初始化由被导入的最深层包开始逐层向外
每个包内部的初始化为**变量初始化->init()->main()**

### 变量初始化

函数作用域内的局部变量初始化顺序：**从左到右、从上到下**

Package 作用域变量：在每一个初始化周期，运行时(runtime)挑选一个没有任何依赖的变量初始化，该过程一直持续到所有的变量均被初始化或者出现依赖嵌套的情形

### Init 函数

> [init functions in Go.](https://medium.com/golangspec/init-functions-in-go-eac191b3860a)

- Init 函数先于 main 函数自动执行，**不能被其他函数调用**
- Init 函数没有输入参数、返回值
- 每个包可以有多个 init 函数，**包的每个源文件也可以有多个 init 函数**，多个 init 函数按照它们的文件名顺序逐个初始化
- 同一个包的 init 执行顺序没有明确定义，编程时要注意程序不要依赖这个执行顺序
- 不同包的 init 函数按照包导入的依赖关系决定执行顺序

golang 对没有使用的导入包会编译报错，可以使用 `import _ packageName` 以仅调用包的 init 函数

Init 函数常用于初始化无法以表达式方式初始化的 package 作用域变量
