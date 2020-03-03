---
bg: 'Gon.png'
layout: post
title:  "golang复合类型"
crawlertitle: "golang"
summary: ""
date:   2019-12-12 23:00:00 +0800
categories: posts
tags: 'golang'
author: 宋天
---

golang的复合类型，包括数组，切片，哈希表和结构体等复合类型的使用




# 数组(Array)
数组，长度固定，包含的元素拥有一种共同类型

- `q := [...]{1,2,3}`初始化的时候可以使用`...`来省略长度，这时长度由后面元素的个数决定

- 数组的长度是它类型的一部分，[3]int和[4]int不是一个类型，数组长度必须是一个常量，也就是必须在编译的时候能确定的值

- 指定标引结合iota定义
 
```go
type Currency int
const (
    USD Currency = itoa
    EUR
    GBP
    RMB
)
symbol := [...]string{USD: "$", EUR: "€", GBP: "£", RMB: "￥"}
```

- 指定特定标引，并给其他值默认值，定义100个值，最后一个是-1，其余是0` r:=[...]int{99: -1}`
- 如果数组的一个元素是可以比较的，那么这个数组类型就是可以比较，可以直接对数组使用==或者!=， 会将两个数组一个一个去比较，但[2]int和[3]int不是一个类型，不能比较
- go的数组当做参数传递的时候会copy一份原数组（对于其他类型go也是这样做的），这样效率比较低，在其他语言中都是隐式的传的引用，在go中可以显示的使用指针来达到同样的效果, 尽管这样，数组也不够灵活，因为他们的长度是固定了，所以很少在func参数使用数组作为参数而是使用切片(slice)


# 切片(Slices)

可变长度，所有元素拥有一种类型，轻量级的数据结构可以访问数组的子序列，包含三部分组成：指针，长度和容量，长度小于等于容量，可以使用len和cap函数获取切片的长度和容量

- s[i:j]切片操作，s可以是一个数组，指向数组的指针或者其他切片，返回一个新的切片， 0≤i≤j≤cap(s), i和j可以单独或者同时省略，i的默认值是0，j的默认值是len(s)
- s[i:j]中j超过cap(s)会报错，超过len(s)会扩展数组。
- substring操作和在[]byte上面的切片是很相似的，都是s[x:y], 都返回原始字节序列的子序列，都依赖潜在数组所以花费常量时间，唯独不同的是，当s是string时返回string,当s时[]byte返回[]byte
- 因为切片含一个指针，当做函数参数传递的时候允许函数去修改底层数组元素，也就是说，拷贝切片创建了底层数组的一个别名
- 和数组不同，切片是不可比较的，对于字节切片，可以使用bytes.Equal来比较，其他类型的切片的比较需要我们自己来实现
- make([]T, len, cap)可以创建一个切片，指定长度和容量，容量可以省略和长度一致。

## append方法

append将一个元素追加到切片s里面，方法内部的机制简单如下，查看s的cap是否够大，如果够大，就使用同样的底层数组，把切片s的len加1，并把元素加入那个位置，如果不够大，就会重新分配内存建一个新的底层数组，然后将数组拷贝过去，并将新元素加入，然后返回新的切片

- 因为我们不能确定每次调用append方法的时候会不会重新分配内存，所以通常我们会使用append的返回值赋值给原来的切片，而不能去依赖append方法以参数引用改变切片。

```
s := append(s, a)  

```

## 一些原位操作
- 这些操作并不会重新分配内存，而是利用原有的底层数组

```
func nonempty(strings []string) []string {
	i := 0
	for _, s := range strings {
		if s != "" {
			strings[i] = s
			i++
		}
	}
	return strings[:i]
}
func remove(s []int, i int) []int  {
	copy(s[i:], s[i+1:])
	return s[:len(s)-1]
}

```


# 哈希表(Map)

- 哈希表是一个无序的键值对，键是唯一的，无论表多大，根据键去检索，更新，删除值平均花费常量的时间。

- Go的map是一个哈希表的一个引用，map[K]V, 键拥有相同的类型，值也拥有相同的类型，键和值的类型可以不同，键必须是可以比较的，这样才能确保键的唯一性，最好不要用浮点作为key, 因为浮点比较不准而且NaN不能去比较

- 使用make或者map字面量来生成map, 使用delete来删除，使用下标来检索

```
m := make(map[string]int)
m := map[string]int{
    "haha":1,
    "heihei":2
}
// delete
delete(m, "haha")

// retrieve
m["haha"]
```

- 获取不了map元素的地址，其中一个原因是因为map重新哈希已有元素，使之前的地址失效
- 使用range来遍历map的k,v, map本身是无序的，但我们可以将key放入一个数组排序，然后按照我们排序的key取出来
- 查询，删除，遍历一个nil的map是安全的，但不是赋值一个nil map
- 和切片一样，map不能直接比较
- 因为map对于不存在的key是取0值的，在检索时使用第二个返回值来判断key是不是真正存在map中，如下

```
if age, ok := ages["bob"]; !ok {}
```

- go没有提供set这样的类型，map的key是唯一的，可以当做set来使用

# 结构体(Struct)

结构体就是有0个或者多个字段的复合类型

- go在我们使用结构体指针的时候做了一些简化操作，不用显示写*也能使用结构体，如下

```go
var employeeOfTheMonth *Employee = &dilbert // dilbert is a Employee type struct, i omit the context

employeeOfTheMonth.Position = "xxx"
// is equivalent to
(*employeeOfTheMonth).Position = "xxx"
```
- 当一个方法返回一个结构体指针的时候，是可以直接方法其属性的，但返回一个结构体的时候不能，如下

```go

type Account struct {
	Name string
}
func NewAccount() Account {
	return Account{}
}
NewAccount().Name = "xxx"
//  error cannot assign to NewAccount().Name


func NewAccount() *Account {
	return &Account{}
}
NewAccount().Name = "xxx"
// pass

```
- 结构体的字段顺序是有意义的，顺序不同则是不同的结构体
- 结构体类型S不能包含结构体类型S, 但可以包含结构体类型指针*S, 这样我们可以去创建一些链表和树式的类型
- 结构体的零值就是每个属性类型的零值
- 空结构体不存数据，所以会有人使用set时，用这样的结构map[string]struct{}, 但是节省空间不重要，而且这种写法很繁琐，最好不要这样写

## 结构体字面量

```go
type Point struct { X, Y int }

//形式一, 简单，但需要记住属性的位置，在增加属性之后也需要调整位置
p := Point{1, 2}
//形式二，显示指明属性名，没指定属性的值为零值
p := Point{X:1}

```
- 结构体可以当作函数的参数传递，但遇到比较大的机构体时，为了提高效率，最好传递指针，而且当函数想要改变传入的参数时必须这么做，因为go只有值传递。
- `pp := &Point{1,2}`创建并初始化一个Point指针

## 比较结构体

- 如果结构体的所有字段都是可比较的，那么结构体就是可比较的
- 可比较的结构体可以作为map的key

## 结构体的内置匿名属性

- 结构体可以通过组合的方式来减少代码的重复性, 但这样的话，需要多个`.`符合来获取到嵌套结构体的属性

```go
type Point struct { X, Y int }
type Circle struct {
    Center Point
    Radius int
}
type Wheel struct {
    Circle Circle
    Spokes int
}

var c Circle
c.Center.X = 1
var w Wheel
w.CirCle.Center.X = 1
```
- 使用匿名字段的方式可以使用一个`.`来获取嵌套结构体的属性

```go
type Point struct { X, Y int }
type Circle struct {
    Point
    Radius int
}
type Wheel struct {
    Circle
    Spokes int
}

var c Circle
c.X = 1
var w Wheel
w.X = 1
```

- 但在使用字面量初始化的时候还需要些多层结构体
- 匿名属性不一定非得是结构体，可以是其他任意被命名的类型或者指向其的指针，匿名属性对于这些类型的意义不在于节省`.`符，而且为了继承类型的方法

## JSON

- json.Marshal(movies) 将go对象转为Json字节数组， json.MarshalIndent(movies, "", "	"), 转为格式可读的Json字节数组，没被导出的字段不会被转化为json
- 字段标签, 是编译时期和结构体属性相关的元数据字符串，里面的json key控制着encoding/json的转化表现

```
Year int `json: "released"`

```
- json.Unmarshal(data, &titles), 将字节序列转化为go对象，这个过程对字段的忽视大小写，所以不用给那些只有大小不同的字段增加字段标签

## 文本和HTML

- text/template和html/template包
- 一些表达式和使用管道符`|`对表达式进行处理