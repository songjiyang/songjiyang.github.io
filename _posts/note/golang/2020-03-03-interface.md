---
bg: 'Gon.png'
layout: post
title:  "golang接口"
crawlertitle: "golang"
summary: ""
date:   2020-03-03 18:30:00 +0800
categories: posts
tags: 'golang'
author: 宋天
---

Golang中的接口，接口是Go中一个重要的概念，类似Java中的Object, 对类型的抽象起到了重要的作用




接口类型是一些其他类型表现的概括和抽象，通过概括，可以让我们写出更灵活和扩展的函数，因为这些函数没有和特定类型绑定在一起


-  go的接口和其他语言不同的是，go是隐式满足的，即只要某个类型拥有了这个接口的方法，就实现了这个接口
-  接口是一种抽象类型，与之对应的是具体类型，抽象类型没有内在的数据结构，只拥有一些方法，通过这些方法可以知道这个接口可以做什么


Fprintf方法的第一个参数就是一个接口, 只有拥有Write的方法的类型就可以传入Fprintf, 例如Printf将标准输出传进去，Sprintf将Buffer内存缓存传进去（go新版这个函数已经变了)

```go
func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)


type Writer interface {
	Write(p []byte) (n int, err error)
}

func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}

func Sprintf(format string, args ...interface{}) string {
    var buf bytes.BufferFprintf(&buf, format, args...)
    return buf.String()
}
```

## 满足接口

当一个具体类型拥有了某个接口类型的所有方法，那么这个具体类型满足了这个接口类型，也能说这个具体类型是这个接口类型

- 接口的赋值规则很简单，只有满足接口类型都可以赋值给接口变量
- 类型T可以调用`*T`的方法，只要参数是个变量（而不是字面量), 因为编译器会隐式取它的地址，不过这仅仅是语法糖，在满足接口中，类型T不会拥有*T的方法，所以也就无法满足某些接口
- 当一个具体类型被赋值给一个接口类型时，只能调用该接口类型拥有的方法，即使这个具体类型还会有其他的方法
- 空接口类型`var any interface{}`， 任何类型都满足它，所以可以将任何类型赋值给它，但因为空接口没有任何方法，所以被赋值的具体类型的方法也不能使用，只有当我们转换回来才能使用

## flag包例子

flag包可以从命令行参数中获取值，或者显示默认值，或者打印使用帮助信息

使用flag需要满足一下接口, String方法特定类型转为string, Set将命令行中的String转为特定类型

```go
type Value interface {
    String() string
    Set(string) err
}

```

## 接口值

接口的值有两块组成，具体类型和那个类型的值，被称为接口的动态类型和动态值


| -     | -   |
| ----- | --- |
| type  | nil |
| value | nil |



- go中的类型是编译时期的，所以实际在接口中存储的是类型描述符，就是类的名称和所拥有的方法

```go
	var w io.Writer
	w = os.Stdout
	w = new(bytes.Buffer)
	w = nil
	fmt.Println(w)
```

- 当实际调用具体类型的方法时，不能直接去调用，必须编译器根据类型描述符生成一些获取Write方法地址的代码才行，接受者(Reciver)是接口动态值的一份拷贝

```go
    w.Write([]byte("hello")
    // 相当于
    os.Stdout.Write([]byte("hello"))
```

- 接口值是可以比较的，都为nil则相等，或者他们的动态类型一样且他们的动态值使用`==`也相等（取决于不同类型的表现)可以认为相等，可以用于map的key和switch，如果具体类型（例如切片）不能比较，那么将会panic
- 可以使用fmt中的%T来输出接口值的动态类型


## 含有nil指针的接口不是nil

nil接口什么都没有，和含有nil指针的接口不同


- 下面这个接口是和nil不等的, 当被调用时，可以产生panic



| -     | -             |
| ----- | ------------- |
| type  | *bytes.Buffer |
| value | nil           |

## 使用sort.Interface进行排序

Go的sort.Sort函数使用sort.Interface进行原位排序(内部有多种排序算法), 实现这个接口就可以传入sort.Sort函数进行排序

```go
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

- sort.Reverse方法返回一个新的Interface覆盖了原Less方法

```go
type reverse struct {
	// This embedded Interface permits Reverse to use the methods of
	// another Interface implementation.
	Interface
}

// Less returns the opposite of the embedded implementation's Less method.
func (r reverse) Less(i, j int) bool {
	return r.Interface.Less(j, i)
}

// Reverse returns the reverse order for data.
func Reverse(data Interface) Interface {
	return &reverse{data}
}


sort.Sort(sort.Reverse(byArtist(tracks)))
```

## error接口

error类型是一个接口类型实现了error接口

```go
type error interface {
    Error() string
}
```

- 创建一个error类型最简单的方式是使用errors.New, 根据提供的错误信息返回一个新的error

## 类型断言

`x.(T)` x是一个接口,当T是一个类型，检查x这个接口的动态类型是不是T类型，如果是的话，将返回T类型的结果，如果不是的话，将产生panic；

当T是一个接口类型时，检查x这个接口的动态类型是不是满足T接口,如果满足的话，将x这个接口类型转换为T接口类型，它的动态类型和值是没变的，这样的话，返回值会暴露不同的方法子集，原来是x接口类型的方法集，后来是T接口类型的方法集


- nil接口，类型断言会失败
- 向方法较少的接口类型断言和赋值没什么区别, 除了x是nil时会失败，赋值不会失败
- 接收类型断言的第二个返回值可以让类型断言失败时，第二个返回值ok为false, 如下

```go
if f, ok := w.(*os.File); ok {
    // ...use f...
}
```

- 可以使用接口类型断言来查询表现，例如断言一个空接口是error或者string， 就知道这个值可以做什么

## 类型选择

接口有两种不同的使用风格，一个是定义了一些方法，具体类型实现了这些方法，接口隐藏了这些具体类型方法的实现，所以这种情况下强调的是方法，而不是具体类型，二是接口持有不同的具体类型，而接口是这些类型的集合，使用类型断言可以从这些类型转换，这种情况强调具体类型实现了接口，而不是接口的方法


- 多个连续用if else的类型断言可以使用，switch来代替，type是关键字

```go
switch x.(type) {
case nil:   // ...
case int, uint: // ...
case bool:   // ...
case string:  // ...
default:    // ...
}
```

## 建议

- 对于只有一个实现的接口是不必要的，它们有运行时代价，有多个实现的时候使用接口才必要, 除了接口和它的实现不在一个包下的情况例外
- 接口越少，越简单的方法越好，更容易被实现
- go支持了面向对象编程，但不是说必须这样，单独的函数，未封装的类型有可以很好的使用