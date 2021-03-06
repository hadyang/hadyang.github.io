
golang 的原生类型十分丰富，除了常见的数值、字符串类型，还包含函数、channel 以及 map 容器。 map 在业务场景下很常见，go 中也可以作为 set 的实现。

那大家是否有想过，什么类型能作为 map 的 key 呢？function、channel、slice 这些类型能作为 key 吗？为解答这个问题，我们先从类型匹配讲起。

## 类型匹配

通过定义类型声明的类型都是不相同的，其他类型可以通过其底层类型是否相同来判断。go 基础类型可以通过以下规则判断是否相同：
- **数组类型**：相同的元素类型 & 相同的数组长度
- **Slice**：相同的元素类型
- **Struct**：相同的字段顺序 & 对应字段有相同的名称、相同的类型、相同的Tag；不同包的未导出字段肯定是不相同的
- **指针类型**：相同的基础类
- **函数类型**：相同个数的参数和返回值 & 对应参数和返回值类型相同，有可变参数也要一致，参数和返回值名称不要求一致。
- **interface**：相同名称和类型的函数的方法集（顺序无关）；不同包的未导出方法肯定是不相同的。
- **map**：两个 map 的 key 和 value 的类型分别相同
- **channel**：相同的元素类型和数据流向

```go
type (
        A0 = []string
        A1 = A0
        A2 = struct{ a, b int }
        A3 = int
        A4 = func(A3, float64) *A0
        A5 = func(x int, _ float64) *[]string
)

type (
        B0 A0
        B1 []string
        B2 struct{ a, b int }
        B3 struct{ a, c int }
        B4 func(int, float64) *B0
        B5 func(x int, y float64) *A1
)

type        C0 = B0
```

以下类型是相同的：

```go
A0, A1, and []string
A2 and struct{ a, b int }
A3 and int
A4, func(int, float64) *[]string, and A5

B0 and C0
[]int and []int
struct{ a, b *T5 } and struct{ a, b *T5 }
func(x int, y float64) *[]string, func(int, float64) (result *[]string), and A5
```

`B0` 和 `B1` 不相同，是因为其都是定义类型声明的类型。`func(int, float64) *B0` 和 `func(x int, y float64) *[]string` 不相同，因为 `B0` 和 `[]string` 不相同。

## 可赋值

在介绍比较操作前，我们还需要了解下什么是 **可赋值**？可赋值就是 变量 `x` 可赋值给类型 `T`。对于不同类型的可赋值条件不一样，当在满足以下条件时，`x`可赋值给类型`T`：
- `x` 的类型与 `T` 相同
- `x` 的类型 `V` 与 `T` 有相同的底层类型，并且 `V` 和 `T` 中至少存在一个非定义类型
- `T` 是接口类型，并且 `x` 实现了 `T`
- `x` 是双向 channel ， `T` 是channel 类型，`x` 的类型 `V` 和 `T` 有相同的元素类型，并且 `V` 和 `T` 中至少存在一个非定义类型
- `x` 是预先声明的 `nil` ，并且 `T` 是指针、函数、slice 、map 、channel 或接口类型
- `x` 是未类型化的 constant 值，并且其类型为 `T`


## 比较操作

对于任意的比较操作，第二个操作数类型对第一个操作数来说必须是 **可赋值** 的，反过来也是一样的。等值操作 `==` 和 `!=` 的两个操作数是 **可比较** 的，比较大小的操作 `<`, `<=`, `>`, 和 `>=` 的操作数是 **可排序** 的。

|        | slice | map  | function | boolean | integer | 浮点数 | 复数 | 字符串 | 指针类型 | channel | interface | 结构体                  | 数组                  |
| :----- | :---- | :--- | :------- | :------ | :------ | :----- | :--- | :----- | :------- | :------ | :-------- | :---------------------- | :-------------------- |
| 可比较 | ❌     | ❌    | ❌        | ✅       | ✅       | ✅      | ✅    | ✅      | ✅        | ✅       | ✅（动态类型可比较时）         | ✅（字段都是可比较的时） | ✅(元素都是可比较的时) |
| 可排序 |       |      |          |         | ✅       | ✅      |      | ✅      |          |         |           |                         |                       |


- 两个复数 `u` 和 `v` 当 `real(u) == real(v)` 并且 `imag(u) == imag(v)` 时，`u` 和 `v` 相等
- 两个指针指向同一个变量或均为 `nil` 时相等；指向不同 'zero-size'（空结构体、空数组） 变量的指针，它们可能相等也可能不相等
- 两个 channel 是由相同的 `make` 调用创建的，或者两个值都为 nil ，它们相等
- 两个 interface 有相同的动态类型和值，或均为 nil ，则它们相同
- 值 `x` 有非接口类型 `X`，值 `t` 有接口类型 `T`，类型 `T` 是可比较的，并且 `X` 实现了 `T`。当 `t` 的动态类型和 `X` 相同，并且 `t` 的动态值与 `x` 相等时，`x` 和 `t` 是相等的。
- 两个结构体的对应非空白标志（由 `_` 标示）相等，则它们相等
- 两个数组中各个对应的元素相等，那么它们相等

## 不可比较导致 panic

`slice`、`map` 和 `function` 是不可比较的，但都可以与预定义的 `nil` 进行比较。当两个接口值的动态类型是 **不可比较** 的，在代码中进行了比较时，会导致 **panic**。这种行为不只是接口值，还包含接口数组或包含接口的结构体。

```go
func main() {
	f1 := make([]int, 0)
	f2 := make([]int, 0)

	var i1 interface{}
	var i2 interface{}
	i1, i2 = f1, f2

	println(i1 == i2) //panic
}
```

对于 map 来说，其 key 必须是 **可比较** 的，因此 `slice`、`map` 和 `function` 均不能作为 map 的key，如果使用则会导致 panic 发生。其他包括 channel、数组在内的类型都能作为 key。

```go
func A() {
}

func main() {
	m := make(map[interface{}]struct{})
	m[A] = struct{}{} //panic

	fmt.Println(m) 
}
```