# Go 泛型


## 1 背景概念

> 这一部分内容 Copy 于文档，可以直接看文档 [**Properties of types and values**](https://go.dev/ref/spec#Properties_of_types_and_values)。

### 1.1 Underlying types

每一种类型都有着对应的 underlying types（潜在类型），基础类型的 underlying type 就是它本身。当你使用 `type` 类重命名类型时，其 underlying type 就是声明的类型了。

```go
type (
	A1 = string
	A2 = A1
)

type (
	B1 string
	B2 B1
	B3 []B1
	B4 B3
)

func f[P any](x P) { … }
```

上面的声明中，`A1` `A2` `B1` `B2` 的 underlying types 都是 `string`，`[]B1` `B3` `B4` 的 underlying types 是 `[]B1`，`P` 的 underlying types 是 `interface{}`。

## 2 泛型基本概念

### 2.1 Type parameter

type parameter（类型参数）指的是表示类型的参数，用于泛型定义中。

例如下面声明中，`P` 就是一个类型参数。

```go
func f[P any](x P) { … }
```

无论是在泛型函数还是泛型类中，都在 `[]` 中指定 type parameter。

type parameter 的声明语法如下：

```go
func f[P any](x P) { … }  // 定义一个 type parameter

func f[M map[K]V, K comparable, V any](m M) { … }  // 定义多个 type paramter，并可以复用
```

### 2.2 Type constraint

type constraint（类型约束）指的是对 type parameter 的约束，表示 type parameter 能够支持哪些类型。

最基本的，type parameter 后跟的类型就是指唯一的 type constraint。

```go
func f[P any](x P) { … }  // [P any] 表示 P 支持任何类型

func f[P int](x P) { … }  // [P int] 表示 P 仅仅支持 int 类型
```

默认情况下，类型约束必须传入的类型完全匹配，使用 `~` 允许 underlying type 匹配即可。

```go
type MyInt int

func f[P int](v T) { /*...*/ }  // 仅仅支持 T 是 int 类型，MyInt 类型不可以

func f[P ~int](v T) { /*...*/ }  // 支持 T 的 underlying type 为 int 类型，MyInt 可以
```

使用 `|` 连接多种支持的类型。

```go
func f[T int|string](v T) { /*...*/ }  
```

interface 现在不仅仅可以包含方法，还可以包含 type constraint，意味着不仅要满足方法要求，还要满足类型要求。

```go
type Number interface {
    ~int64 | ~float64
}

func f[N Number](nr Number) { /*...*/ }  // 等价与 [N ~int64|~float64]

type NumberWithString interface {
    ~int64 | ~float64

    String() string
}

type MyIntWithString int64  // implement NumberWithString

func(m MyIntWithString) String() string {  /*...*/ }

// 仅仅允许实现了 NumberWithString，并且满足 type constraint 的类型
func f[N Number](nr NumberWithString) string { 
    return nr.String()  // 允许使用 interface 的方法
}  
```

{{< admonition note Note>}}
interface 添加了 type constraint 后，就只允许用作 type constraint 了，不允许用于声明变量了。
{{< /admonition >}}

## 3 使用泛型

### 3.1 Generic function

函数定义中加上 type parameter 就表示 generic function，在编译期会对不同类型的调用生成对应函数。

```go
func min[T ~int|~float64](x, y T) T {
	if x < y {
		return x
	}
	return y
}
```

调用 generic function 时如果传入的参数可以被编译器确定类型，那么就可以省略指定 type parameter。

```go
func main() {
	fmt.Println(min[int](1, 3))  // 指定 type parameter

	fmt.Println(min(1.3, 3.4))  // 省略 type parameter，编译器自动推导
}
```

### 3.2 Generic type

类型定义中加上 type parameter 定义一个 generic type，编译期生成不同的类型。

```go
type List[T any] struct {
	next  *List[T]
	value T
}
```

在 generic type 的方法中，也必须是声明相同数量的 type parameter，命名可以不同。

```go
func (l *List[T]) Len() int  { /* ... */ }
```

同样，在任何使用该类型的地方，都必须传入 type parameter 以指定类型。

```go
func PrintList[T any](list List[T]) { /* ... */ }  // 必须声明 type parameter 以传递给 List

type Printer[T any] interface { // 必须声明 type parameter 以传递给 List
	Print(List[T])
}
```

### 3.3 实例化

所有带有 type parameter 的 type/function/interface 都无法直接使用，必须实例化后才能使用。也就是说，只有给 type parameter 指明类型后才称为 type/funciton/interface。

```go
func Print

type Printer[T any] interface { 
	Print(v T)
}

type IntPrinter[T any] struct {}

func (p *IntPrinter[T]) Print(v T) {}

func Print[T any](v T) {}

func main() {
	var p *IntPrinter  // illegal: IntPrinter 类型没有指明 type parameter
	var iface Printer = &IntPrinter[int]{}  // illegal: Printer 接口没有指明 type parameter
	fn := Print  // illegal: Print 函数没有指明 type parameter


	var p *IntPrinter[int]
	var iface Printer[int] = &IntPrinter[int]{}  // ok
	fn := Print[int]  // ok
}
```

## 4 相关的类型与库

### 4.1 类型

新添加了一些内置类型：

* **`any`** - `interface{}` 的别名。
* **`comparable`** - interface 实现的 type constraint（因此只能用于 type parameter），用于内置的所有允许使用 `==` 或 `!=` 比较的类型。

### 4.2 库

新添加了一些实现性的库：

* `golang.org/x/exp/constraints` - 一些通用的 type constraint，例如 constraints.Float 约束所有 float 类型。
* `golang.org/x/exp/slices` - 针对于 slice 的 generic function。
* `golang.org/x/exp/maps` - 针对于 map 的 generic function。

## 参考

* [**官方文档：Golang 语法规范**](https://go.dev/ref/spec)
* [**Go 1.18 Release Notes**](https://go.dev/doc/go1.18)
