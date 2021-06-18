
## 类型声明

Go 中有两种类型声明是将一个确定的类型绑定到另一个类型上，有两种声明的方法，分别是 **别名声明** 和 **类型定义**

```go
//别名声明
type (
        nodeList = []*Node  // nodeList 和 []*Node 是相同类型
        Polar    = polar    // Polar 和 polar 是相同类型
)
//类型定义
//通过一个底层类型，创建一个新的唯一的类型
type (
	Point       struct{ x, y float64 } // Point 和 struct{ x, y float64 } 不同类型
	polar       Point                  // polar 和 Point 不同类型
	MyInterface cipher.Block           // MyInterface 继承 cipher.Block 方法集
	TPoint      struct{ Point }        // TPoint 继承 Point 方法集
)
```

在类型定义形式中，除接口类型定义、组合定义外，其他定义方式均 **不继承** 方法集。

## Bool类型

bool 类型代表一组 boolean 值，通过预定义的 `true` 和 `false` 来表示。

## 数值类型

数值类型代表一组整数或浮点数，包含： `intx` , `uintx`, `floatx`, `complex64`, `complex128`, `byte`, `rune`, `uintptr`；同时 Go 中存在预定义的与具体平台实现相关的类型：

```go
uint     //可能是32位也可能是64位
int      //大小与 uint 一致
```

`uintptr` 是无符号的整数，其大小足够容纳当前机器上的所有指针范围。数值类型中，除 `rune` 与 `int32` 、`byte` 与 `uin8` 之外，其他均不是一种类型。当不同数值类型在同一个表达式或赋值语句中时，需要进行明确的类型转换。比如说， `int32` 和 `int` 即使在某些机器上有相同的大小，但仍然不是同一种类型。

## String类型

`string` 类型代表一组字符串，字符串（可能为空）由一个字节序列构成。字节序列的长度就是该字符串的长度，并且不会为负数。字符串是 **不可变** 的，一旦创建，无法修改字符串的内容。

可以通过内置的 `len` 函数计算字符串 `s` 的长度，当字符串是常量时，该字符串的长度也是编译时的常量。字符串的字节序列可以通过 `0～len(s)-1` 的整数来访问，获取字节序列中元素的地址是 **非法** 的。如果 `s[i]` 是字符串的第 `i` 个字节, `&s[i]` 是错误的。

## Array类型

数组类型是单一元素类型的编号序列（**固定长度**）。数组的长度就是元素的个数，且不会为负数。

数组的长度也是其类型的一部分（**不同长度的数组属于不同类型**），长度必须是可非负数的 `int` 常量。数组 `a` 的长度，可以通过内置的 `len` 函数获取。数组中的元素可以通过 `0～len(s)-1` 的整数来访问。数组类型通常都是一维的，可以通过组合数组来实现多维类型。

```go
[32]byte
[2*N] struct { x, y int32 }
[1000]*float64
[3][5]int
[2][2][2]float64  // same as [2]([2]([2]float64))
```

## Slice 类型

slice 是底层数组中一个连续段的描述符，提供访问数组的能力。一个 slice 类型表示其元素类型的所有数组 slice 的集合。未初始化的 slice 的值为 `nil`。

slice 的长度表示其内部元素的个数，并且可以由内置的 `len` 函数获取，与数组不同的是，slice 的长度是可以在运行时改变的。slice 中的元素可以通过 `0～len(s)-1` 的整数来访问。

slice 一旦被初始化，就始终会关联一个持有其元素的底层数组。因此，slice 会与关联到其底层数组的所有 slice **共享内存** 的，相对而言，不同的数组会有不同的内存空间。slice 可能是底层数组中间的一段，底层数组的大小可以通过内置的 `cap` 函数获取。

可以通过 `make` 函数创建一个新的、已初始化的类型为 `T` 的 slice。通过 `make` 函数创建的 slice 始终会创建一个新的、隐藏的 **关联数组**。以下两个表达式是等价的：

```go
make([]int, 50, 100)
new([100]int)[0:50]
```

与数组类似，slice 始终都是一维的，但是可以通过组合多个 slice 得到多维的。多维的 slice 的长度也是动态的，并且内部的 slice 必须 **逐个初始化**。

## Struct类型

一个 struct 是一组命名元素（字段）的序列，每个元素都有其名称和类型。字段的名称可以是现式指定的，也可以是隐式的（嵌套）。在一个结构体中，非空白标志字段的名称必须是唯一的。

```go
// An empty struct.
struct {}

// A struct with 6 fields.
struct {
        x, y int
        u float32
        _ float32  // padding
        A *[]int
        F func()
}
```

如果一个字段声明了类型，但是没有现式地指定名称，称之为嵌套字段。嵌套字段必须指定类型 `T`，或者指向非接口的指针类型 `*T`，并且 T 可能不是个指针类型。

```go
// 以下声明是正确的
struct {
        T1        // field name is T1
        *T2       // field name is T2
        P.T3      // field name is T3
        *P.T4     // field name is T4
        x, y int  // field names are x and y
}
//以下声明是非法的，因为其中字段名称不是唯一的。
struct {
        T     // conflicts with embedded field *T and *P.T
        *T    // conflicts with embedded field T and *P.T
        *P.T  // conflicts with embedded field T and *T
}
```

struct `x` 中的某个嵌套字段有方法或字段 `f` ，如果 `x.f` 是合法的表达式并且代表方法或字段 `f`，那么就称该方法或字段 `f` 提升到 `x`。提升字段的行为与原始字段一致，除非提升字段的名称与其他字段有冲突。

## 指针类型

针对基础类型 `T` ，指针类型 `*T` 代表指向 `T` 变量的指针集合，未初始化的指针值为 `nil`。

## 函数类型

一个函数类型，代表拥有相关参数和返回类型的函数集合，未初始化的函数类型为 `nil`。对于函数类型中声明的参数和返回类型，要么都定义变量名，要么都没有变量名。

```go
//fail
type MyF func(x string, int) error 
type MyF func(x string, x int) error  //如果定义了变量名，除空白标志外均需唯一

//pass
type MyF func(x string, y int) error
type MyF func(x string) error
```

## Interface 类型

一个接口类型定义一个方法集，类型 `T` 的方法集是接口 `I` 定义方法集的子集，则 `T` 实现了接口 `I`。类型 `I` 声明的变量可以存储类型 `T` 的值，未初始化的接口类型为 `nil`。

go 中的接口实现是隐式的，并且根据方法集的子集来判定，因此一个类型可能实现多个接口，比如，所有类型都实现了接口类型 `interface{}`。

接口类型除了直接声明方法外，还能通过组合其他接口来定义方法集，接口类型的方法集是组合接口和显式定义接口的 **并集**。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
	Close() error
}
type Writer interface {
	Write(p []byte) (n int, err error)
	Close() error
}
// ReadWriter 的方法集是 Read, Write, Close.
type ReadWriter interface {
	Reader
	Writer
        Close() error
}

// 编译失败
type ReadWriter interface {
	Reader
	Writer
        Close() //如果方法名称有重复，那么其方法签名必须一致
}
```

## map 类型

map 是由唯一Key和其对应元素构成的二元组集合。map 是无序的，未初始化的值为 `nil`。

map 之间的 `==` 和 `!=` 操作由其 Key 的类型决定，因此，函数类型、 map 类型或者 Slice 类型作为 Key 时，都 **不能** 进行比较。如果 Key 的类型是 `interface{}` ，那么 Key 的具体类型也必须是可比较的。

map 中元素的个数称为其长度，可以通过内建的函数 `len` 来获取，元素可以通过 `delete` 函数删除 map 中的元素，`make` 函数可以创建一个特定元素类型以及容量的 map。

## channel 类型

channel 类型通过 `send` `receive` 特定类型元素的方式，为并发执行的函数提供消息传递的能力，未初始化的值未 `nil`。

`<-` 操作符可以操作 channel 来发送或接收消息，也能定义 channel 的方向，无 `<-` 的 channel 可双向操作。channel 可以通过声明或显式转换，构建单向的 channel。

```go
	var c chan int
	var cr <-chan int //只能进行接收操作的 channel
	cr = c
```

通过 `make` 函数可以创建特定类型和容量的新 channel，如果容量为 0 或不指定，则为 **无缓存** channel，只有当发送者和接收者都就绪时，才能完成消息传递。对于带缓存的 channel 来说，发送者（channel 未满）和接受者（channel 非空）可以无阻塞的操作 channel。

内建函数 `close` 可以关闭 channel，接受者可以通过多返回值来判断 channel 是否关闭。

```go
	c = make(chan int)
	if v, ok := <-c; ok {
		println(v)
	}
```

一个 channel 可以在多个 goroutine 中无需同步的调用 `send` 、 `receive` 、`len` 、`cap` 操作。同时，channel 是 **FIFO** 队列。