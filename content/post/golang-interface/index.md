---
title: 【Golang进阶】结构体与方法
draft: true
date: 2021-04-10
categories: 
    - Go
---

在之前的 [《【Golang进阶】函数与栈》]({{< ref "/post/golang-func-stack" >}})文章中，我们通过汇编学习了 Go 的栈结构。今天，我们也通过汇编再来看看结构体和方法调用的实现。

## 结构体初始化

首先，我们定一个结构体 `Dog` 以及一个指针接收者的方法 `Say`。

```go
type Dog struct {
	Name string
	Age  int
}

func (t Dog) Sleep() string {
	return t.Name
}

func (t *Dog) Say() string {
	return t.Name
}

func main() {
	var d1 = &Dog{"煎饼", 43}
	a := d1.Say()
	b := d1.Sleep()
	_, _ = a, b
}
```

执行命令 `go tool compile -S -l -N main.go` 得到汇编代码，代码的 13 行进行 Struct 的初始化，并获取其地址。 `Dog` 包含两个成员变量，分别是 `string` 类型和 `int` 类型。 `int` 类型在底层结构中，是原生表示，但 `string` 是由一个结构体 `reflect.StringHeader` 表示的。

```go
type StringHeader struct {
	Data uintptr
	Len  int
}
```

可以看到 `StringHeader` 有两个变量，`Data` 指向内存中的一个地址， `Len` 表示字符串的长度（len 函数执行的结果），下面就来看看初始化的逻辑。

```nasm
MOVQ	$0, ""..autotmp_4+112(SP)       ;StringHeader.Data=0
XORPS	X0, X0                          ;X0=0
MOVUPS	X0, ""..autotmp_4+120(SP)       ;StringHeader.Len=X0
LEAQ	""..autotmp_4+112(SP), AX       ;AX=&StringHeader.Data
MOVQ	AX, ""..autotmp_3+48(SP)        ;
TESTB	AL, (AX)                        ;nil check
MOVQ	$8, ""..autotmp_4+120(SP)       ;StringHeader.Len=8
LEAQ	go.string."WangWang"(SB), CX    ;CX=&"WangWang"
MOVQ	CX, ""..autotmp_4+112(SP)       ;StringHeader.Data=CX
```

初始化完 Dog 的 `Name` 变量后，接着就是初始化 `Age` ，由于 int 类型本身很简单，因此其初始化过程也简单得多。

```nasm
MOVQ	$43, ""..autotmp_2+56(SP)       ;Dog.Age=43
MOVQ	AX, "".d1+24(SP)                ;*d1=AX
```

在上面的汇编代码中，有几个比较特殊的指令，下面一起来说明下：

1. `XORPS	X0, X0` 异或指令，这里仅仅是将 `X0` 寄存器置 0 （X0是浮点数寄存器）
2. `TESTB	AL, (AX)` 逻辑与指令，这里是用做 nil check，如果加载 `AX` 失败会触发段错误信号 `SIGSEGV`，触发 Go Runtime 抛出 Panic。选择 `TESTB` 仅仅是因为指令短小。
3. `go.string."煎饼"(SB)` 重定向符号，这种带 `(SB)` 的都是用户获取重定向符号的地址

关于重定向，我们这里再看下上面的代码， `go.string."煎饼"(SB)` 在 `main` 函数中作为重定向符号引用，其具体地址会在 Link 阶段确定。

```
go.string."煎饼" SRODATA dupok size=6
	0x0000 e7 85 8e e9 a5 bc                                ...... ; 这里的二进制就是 "煎饼" 的字面值
```

上面的汇编代码就是符号的定义，`SRODATA` 表示只读数据， `DUPOK` 表示单个二进制文件中可以有多个定义，但在 Link 阶段该符号会唯一对应一个地址。

## 结构体方法调用

### 指针接收者


结构体的初始化完成，下面我们就来看看指针接收者的函数调用。

```nasm
MOVQ	AX, (SP)            ;SP=d1
CALL	"".(*Dog).Say(SB)   ;调用 Say 函数，并且 SP=SP-8

;下面是函数 Say 的汇编代码
XORPS	X0, X0              ;X0=0
MOVUPS	X0, "".~r0+16(SP)   ;r0.Data=X0
MOVQ	"".t+8(SP), AX      ;AX=d1
TESTB	AL, (AX)            ;nil check
MOVQ	(AX), CX            ;CX=d1.Data
MOVQ	8(AX), AX           ;AX=d1.Len
MOVQ	CX, "".~r0+16(SP)   ;r0.Data=CX
MOVQ	AX, "".~r0+24(SP)   ;r0.Len=AX
RET                         ;
```

在指针接收者的调用中，通过 `AX` 寄存器将接收者传递给被调用函数。也可以清晰的看到， Go 在进行函数调用时都使用 **值拷贝** 的方式，只不过指针的值拷贝还是指针。

### 值接收者

相比于指针接收者函数，值接收者函数在调用前会进行更多的复制，占用 **更多** 的指令和栈空间。

```nasm
MOVQ	"".d1+40(SP), AX            ;
TESTB	AL, (AX)                    ;
MOVQ	(AX), CX                    ;
MOVQ	8(AX), DX                   ;
MOVQ	16(AX), AX                  ;
MOVQ	CX, ""..autotmp_5+88(SP)    ;
MOVQ	DX, ""..autotmp_5+96(SP)    ;
MOVQ	AX, ""..autotmp_5+104(SP)   ;
MOVQ	CX, (SP)                    ;复制 d1.StringHeader.Data
MOVQ	DX, 8(SP)                   ;复制 d1.StringHeader.Len
MOVQ	AX, 16(SP)                  ;复制 d1.Age
CALL	"".Dog.Sleep(SB)            ;
```

可以看到即使 `d1` 是指针，Go 在编译时也会转化为值进行调用，并且 Go 会把接收者的数据拷贝一遍，当作参数传递给被调用函数。


## 接口初始化

接口的初始化有两种情况，分别是空接口 `interface{}` 和非空接口，空接口的实现是 `runtime.eface`，包含一个类型字段和一个指向底层数据的指针；非空接口的实现是 `runtime.iface`，包含 `itab` 类型的接口信息数据和一个指向底层数据的指针。

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

定义接口 `Animal` 以及以 `Animal` 为入参的 `Night` 函数。

```go
type Animal interface {
	Sleep() string
	Say() string
}
func main() {
	var d1 = &Dog{"WangWang", 43}
	var t1 Animal = d1
	var t2 interface{} = d1

	_ = t2
	Night(t1)
}
func Night(t Animal) {
	t.Sleep()
}
```

通过汇编可以发现，非空接口中的 `itab` 信息是直接从重定向符号中获取，并且在编译时生成。

```nasm
LEAQ	go.itab.*"".Dog,"".Animal(SB), CX   ;CX=&(go.itab.*"".Dog,"".Animal)
MOVQ	CX, "".t1+56(SP)                    ;t1.itab=CX
MOVQ	AX, "".t1+64(SP)                    ;t1.data=AX
```

空接口的 `_type` 同样也是从重定向符号中获取，编译时生成。其他操作与非空接口类型。

```nasm
LEAQ	type.*"".Dog(SB), CX                ;CX=&(type.*"".Dog)
MOVQ	CX, "".t2+40(SP)                    ;t1._type=CX
MOVQ	AX, "".t2+48(SP)                    ;t1.data=AX
```


## 接口方法调用

接下来，我们再看看接口的方法调用，相比于结构体会更复杂。定义接口 `Animal` 以及以 `Animal` 为入参的 `Night` 函数。




在 `t1` 赋值完成后，就会把 `t1` 拷贝参数并调用 `Night` 方法。并通过 `itab` 定位方法的地址， `itab` 的结构由 `runtime.itab` 表示。

```go
type itab struct {
	inter *interfacetype    //
	_type *_type            //
	hash  uint32            //offset=16byte
	_     [4]byte           //内存对齐
	fun   [1]uintptr        //offset=24byte 方法指针
}
```



```nasm
MOVQ	"".t+40(SP), AX     ;AX=t1.itab
TESTB	AL, (AX)            ;nil check
MOVQ	32(AX), AX          ;AX=t1.itab+32
MOVQ	"".t+48(SP), CX     ;
MOVQ	CX, (SP)            ;
CALL	AX                  ;
```


## 方法集的奥秘

在 `main.go` 文件中，我们还能看到 `"".(*Dog).Sleep STEXT ...` 的声明，但我们的代码并没有指针接收者的 `Sleep` 方法，这就是 Go 编译器自动生成的代码。通过汇编我们也能知道「`*T` 的方法集：所有声明以 `T` 或 `*T`为接收者的方法集合」这条规则的底层原因。

```nasm
TESTB	AL, (AX)
MOVQ	(AX), CX
MOVQ	8(AX), DX
MOVQ	16(AX), AX
MOVQ	CX, ""..autotmp_3+56(SP)
MOVQ	DX, ""..autotmp_3+64(SP)
MOVQ	AX, ""..autotmp_3+72(SP)
MOVQ	CX, (SP)
MOVQ	DX, 8(SP)
MOVQ	AX, 16(SP)
PCDATA	$1, $1
CALL	"".Dog.Sleep(SB)
```

上面是截取的 `"".(*Dog).Sleep` 一段汇编，可以发现其结构与上一节 “值接收者” 中的汇编类似，都是通过指针 **转换为值** 并调用值接收者方法的策略。

那这里我们就可以思考下，既然可以通过代码生成来扩充接收者的方法集，那么能自动生成指针接收者的方法吗？

答案是不行。原因也很简单，值接收者是接收拷贝数据的，并不能找到原始数据的具体地址，因此也无法生成指针接受者。




## 参考文档

[directives](https://golang.org/doc/asm#directives)