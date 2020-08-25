---
title: Golang 方法集
draft: true
date: 2021-03-25
categories: 
    - Go
---


## 方法集

每个类型都有其关联的方法集，没有定义方法的类型关联“空方法集”。方法集中的每个方法名必须是唯一的，并且有类型作为接受者。如果，一个类型的方法集是某个接口方法集的子集，则认为该类型实现了该接口。

1. `T` 的方法集：所有声明以 `T` 为接受者的方法集合
2. `*T` 的方法集：所有声明以 `T` 或 `*T`为接受者的方法集合
3. 如果 struct `S` 包含嵌套字段 `T`，`S` 的方法集包含 receiver 为 `T` 的方法集合
4. 如果 struct `S` 包含嵌套字段 `T`，`*S` 的方法集包含 receiver 为 `T` 和 `*T` 的方法集合
5. 如果 struct `S` 包含嵌套字段 `*T`，`S` 和 `*S` 的方法集包含 receiver 为 `T` 和 `*T` 的方法集合


用 `T` 或者 `*T` 调用方法，不受方法集的约束，编译器可查找所有方法，并自动转换 receiver 类型。

```go
type T int64

func (t T) Write(p []byte) (n int, err error) {
	return 0, err
}

func (t *T) Close() error {
	return nil
}

func main() {
	t := T(1)
	var wc io.WriteCloser
	//wc = t  //编译失败
	wc = &t
	_ = t.Close()

	_ = wc.Close()
}

```

上面这个例子就可以很好的解释方法集和接受者的关系，只有类型 `*T` 实现接口 `io.WriteCloser`，并且两种类型的方法调用不受方法集的影响。
