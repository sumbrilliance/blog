---
title:  golang 中的反射
date: 2019-10-20T21:51:01+08:00
coverImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/home-bg-o.jpg"
thumbnailImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/home-bg-o.jpg"
thumbnailImagePosition: "left"
draft: true
categories:
- 基础
tags: 
- 其他
---

反射是高级语言中常见的特性，golang也不例外。本以为没什么特别的，项目中也很少用到，今天有（同样是曾做过iOS开发的）朋友问我，golang中的反射，是否能像 oc runtime 一样，通过类似ClassFromString，获取实例的类型，并生成新的实例？顿时语塞，故重新温习了一遍golang的reflect，并做记录。本文介绍的为golang中的反射基础概念。

<!--more-->

[toc]

# 什么是反射

大多数时候，我们需要一个类型，那么，

直接声明一个完事：

```go
type User struct {
	Name   string
	Age    int
}
```

需要变量，直接声明一个：

```go
var u User
```

需要 函数，同样声明一个：

```go
func DoSomething(u User) {
	fmt.Println("name = ",u.Name)
	fmt.Println("age = ", u.Age)
}
```

但有时候，我们希望在运行期间，获取到一个变量的类型，和这个类型的一些特性，如包含的属性，含有的方法，而这些，是在编译期无法确定的。例如，从文件或者网络请求中获取到的数据，希望映射到某个实例中。又比如，希望编写的代码，广泛适用与不同的类型（譬如说“泛型”）。这个时候，就需要使用反射特性。它可以让你在运行时修改和创建变量、function和 structs。

# type、kind、和 value

go 的反射，是围绕着三个概念构建：types, kinds, values。

## type

通过 `vatType := reflect.TypeOf(var)` 可以获得某实例变量的 type。返回类型是 reflect.Type 类型。

### Name()

调用方法 .Name()，返回的是该实例的所属类型，例如，在 main 包中定义的类型:

```go
type User struct {
	Name 	string
	Age     int
}
func main() {

	u := &User{
		Name: "张三",
		Age:  12,
	}
	t := reflect.TypeOf(u)
	fmt.Println("type = ", t) // type = *main.User
  fmt.Println("type's name = ", t.Name()) // type's name = 

```

咦，为什么 t.Name() 是空的？

因为 指针 和 slice 类型，通过 Name() 方式是获取不到名字的。如果将 

```go
func main() {

	u := User{
		Name: "张三",
		Age:  12,
	}
	t := reflect.TypeOf(u)
	fmt.Println("type = ", t) // type = main.User
	fmt.Println("type's name = ", t.Name()) // type's name =  User
}
```

则可使用 Name() 获取到名字，可以看到，直接打印 type，是含有包名的，而 t.Name()没有

### static type 和 concrete type

使用 reflect.Typeof() 的一个重要作用，是他可以获得变量的实际类型。

例如

```go
var w io.Writer = os.Stdout
fmt.Println(reflect.TypeOf(w)) // "*os.File"
```

可以明确获取到 w 其实是 File。这是因为 type 可以分为 static type 和 concrete type。static type，就是我们编程时看到的类型（string、int），而 concrete type，是runtime 系统看见的类型，也就是所谓的动态类型。

什么意思呢？看代码：

```go
type I interface{ F() }
type S struct{}
func (S) F() { } // 都实现了 I 接口

type T struct{}
func (T) F() { } // 都实现了 I 接口
var x I
x = S{} // static type 是 I，concrete type 是 S

x = T{} // static type 仍然是 I，但 concrete type 已经是 T
```



## kind

kind 和 type，在中文里都可以翻译成种类，但在这里，kind 和 type 其实有很大的不同。

kind，是指更抽象的类,或者说是内置的基本类型，比如说，如果你定义了一个struct 叫 User，那么，kind就是struct，type是User。进到reflect包 type.go 文件，可以看到 kind 类型，其实是内部类型：

```go
type Type interface {
  ...
  Kind() Kind
  ...
}
type Kind uint
const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	Uint
	Uint8
	Uint16
	Uint32
	Uint64
	Uintptr
	Float32
	Float64
	Complex64
	Complex128
	Array
	Chan
	Func
	Interface
	Map
	Ptr
	Slice
	String
	Struct
	UnsafePointer
)
```

## 示例

通过一个例子来看反射包中常使用的方法：

```go

type User struct {
	Name 	string `tag1:"First Tag" tag2:"Second Tag"`
	Age     int
}

func examiner(t reflect.Type, depth int) {
	fmt.Println(strings.Repeat("\t", depth), "Type is", t.Name(), "and kind is", t.Kind())
	switch t.Kind() {
	case reflect.Array, reflect.Chan, reflect.Map, reflect.Ptr, reflect.Slice:
		fmt.Println(strings.Repeat("\t", depth+1), "Contained type:")
		examiner(t.Elem(), depth+1)
	case reflect.Struct:
		for i := 0; i < t.NumField(); i++ { // struct 类型获取里面的键值
			f := t.Field(i)
			fmt.Println(strings.Repeat("\t", depth+1), "Field", i+1, "name is", f.Name, "type is", f.Type.Name(), "and kind is", f.Type.Kind())
			if f.Tag != "" {
				fmt.Println(strings.Repeat("\t", depth+2), "Tag is", f.Tag)
				fmt.Println(strings.Repeat("\t", depth+2), "tag1 is", f.Tag.Get("tag1"), "tag2 is", f.Tag.Get("tag2"))
			}
		}
	}
}


func main() {
	u := User{
		Name: "zhangsan",
		Age:  12,
	}

	sl := []int{1, 2, 3}
	greeting := "hello"
	greetingPtr := &greeting
	up := &u

	slType := reflect.TypeOf(sl)
	gType := reflect.TypeOf(greeting)
	grpType := reflect.TypeOf(greetingPtr)
	uType := reflect.TypeOf(u)
	upType := reflect.TypeOf(up)

	examiner(slType, 0)
	/*
	 Type is  and kind is slice
	         Contained type:
	         Type is int and kind is int

	*/
	examiner(gType, 0)
	/*
	 Type is string and kind is string
	*/
	examiner(grpType, 0)
	/*
		 Type is  and kind is ptr
	         Contained type:
	         Type is string and kind is string

	*/
	examiner(uType, 0)
	/*
		 Type is User and kind is struct
	         Field 1 name is Name type is string and kind is string
	                 Tag is tag1:"First Tag" tag2:"Second Tag"
	                 tag1 is First Tag tag2 is Second Tag
	         Field 2 name is Age type is int and kind is int


	*/
	examiner(upType, 0)
	/*
	 Type is  and kind is ptr
	         Contained type:
	         Type is User and kind is struct
	                 Field 1 name is Name type is string and kind is string
	                         Tag is tag1:"First Tag" tag2:"Second Tag"
	                         tag1 is First Tag tag2 is Second Tag
	                 Field 2 name is Age type is int and kind is int

	*/
}


```

# reflect.Value - 利用反射，生成实例













### reflect.Type 常用属性总结

- 指针 和 chan 可视作容器类型，可通过 t.Elem() 获取指向的类型。t.Elem() 返回的也是 reflect.Type 类型

- reflect.Type 中的各方法属性代表意义：

  - method(方法) 相关：
    
  - Method(i) Method  获取方法列表中的第 i 个方法
    
    - 
    
  - t.NumField()  - 结构体的 filed 个数，如果 "t" 不是struct类型，会引发panic

  - t.Field(i) - struct 中的第几个 filed，返回 StructField 类型，该类型结构如下

    ```go
    	// Name is the field name. 如举例的 User 中的 "Name" 和 "Age"
    	Name string
    	// PkgPath is the package path that qualifies a lower case (unexported)
    	// field name. It is empty for upper case (exported) field names.
    	// See https://golang.org/ref/spec#Uniqueness_of_identifiers
    	PkgPath string
    	Type      Type      // field type，又是一个 reflect.Type 类型，
    	Tag       StructTag // field tag string，StructTag 也就是String的别名，
    /*
    这里是示例中的 `tag1:"First Tag" tag2:"Second Tag"`,注意，是反引号中的全部字符串，要获取单个tag，使用  f.Tag.Get("tag1")| f.Tag.Get("tag2") 指定tag名获取。
    */
    	Offset    uintptr   // offset within struct, in bytes，在 struct 中的偏移量
    	Index     []int     // index sequence for Type.FieldByIndex， 这个field在struct中所处的下标位置
    	Anonymous bool      // is an embedded field，这个 field 是否是匿名的，例如 
    /*
     type Admin struct {
     		User
     		Passwd String
     }
     中的 User ，就是一个匿名的filed
    */
    ```

- 