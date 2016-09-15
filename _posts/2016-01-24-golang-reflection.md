---
layout: post
title: Golang Reflection
categories: [golang]
tags: [golang, reflection]
---

最近的一个task是要读取环境变量中的配置，于是想到了反射机制。反射机制常常能提供更高维度的视野，可以写出更general的程序。

"reflect"包下主要是Type和Value两个struct：
- Type封装了“类型”的属性，定义相关的东西找他；
- Value主要封装了“值”的属性，与值相关的东西找他没错。此外，他是线程安全的（或者叫goroutine安全）.

# Structs for Case

``` go
type Class struct {
    Name    string   `json:"name"`
    Student *Student `json:"student"`
    Grade   int      `json:"grade"`
    school  string   `json:"school"`
}

type Student struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}
```

# Cycle

反射机制主要是了解它的整个cycle，才能玩的转。。。

## Object->Reflect->Object

``` go
s := Student{Name: "LiLei", Age: 20}

typ := reflect.TypeOf(s)
val := reflect.ValueOf(s)
fmt.Printf("The type is %s.\n", typ.String())
fmt.Printf("The name is %s.\n", val.FieldByName("Name").String())

if s, ok := val.Interface().(Student); ok {
    fmt.Printf("The student is %s.\n", s.Name)
} else {
    fmt.Println("Wrong!")
}

// output:
// The type is main.Student.
// The name is LiLei.
// The student is LiLei.
```

## Type->Object
毕竟golang没有jvm那种东西，不能runtime加载。所以type还是得从hard code得到

``` go
t := reflect.TypeOf(Student{})
val := reflect.New(t)
fmt.Println(val.Type().String())

// output
// *main.Student
```

这里reflect.New(reflect.Type)返回的是指向new出的value的指针。

# Reflect Operation
使用反射最主要的还是要能操作对象啦

## Traverse Object

``` go
s := &Student{"LiLei", 18}
c := &Class{"Class A", s, 6, "Century Ave"}

val := reflect.ValueOf(c)
typ := reflect.TypeOf(c)
if val.Kind() == reflect.Ptr {
    fmt.Printf("It is a pointer. Address its value.\n")
    val = val.Elem()
    typ = typ.Elem()
}

for i := 0; i < val.NumField(); i = i + 1 {
    fv := val.Field(i)
    ft := typ.Field(i)
    switch fv.Kind() {
    case reflect.String:
        fmt.Printf("The %d th %s types %s valuing %s with tag env %s\n", i, ft.Name, "string", fv.String(), ft.Tag.Get("env"))
    case reflect.Int:
        fmt.Printf("The %d th %s types %s valuing %d with tag env %s\n", i, ft.Name, "int", fv.Int(), ft.Tag.Get("env"))
    case reflect.Ptr:
        fmt.Printf("The %d th %s types %s valuing %v with tag env %s\n", i, ft.Name, "pointer", fv.Pointer(), ft.Tag.Get("env"))
    }
}

// It is a pointer. Address its value.
// The 0 th Name types string valuing Class A with tag env NAME
// The 1 th Student types pointer valuing 826814776864 with tag env STUDENT
// The 2 th Grade types int valuing 6 with tag env GRADE
// The 3 th school types string valuing Century Ave with tag env SCHOOL
```

这里，私有的属性也是能遍历到值的。Tag可以为struct附带很多信息，合理利用可以出奇迹啊。

## Modify Object

``` go
c := &Class{}

val := reflect.ValueOf(c).Elem()
typ := reflect.TypeOf(c).Elem()

for i := 0; i < val.NumField(); i = i + 1 {
    fv := val.Field(i)
    ft := typ.Field(i)
    if !fv.CanSet() {
        fmt.Printf("The %d th %s is unaccessible.\n", i, ft.Name)
        continue
    }

    switch fv.Kind() {
    case reflect.String:
        fv.SetString("LiLei")
    case reflect.Int:
        fv.SetInt(18)
    case reflect.Ptr:
        continue
    }
}

fmt.Printf("%v\n", c)

// output:
// The 3 th school is unaccessible.
// &{LiLei <nil> 18 }
```

## Map/Array/Slice/Channel

Golang的reflect还针对其他几个类型提供了特殊的api。

``` go
m := map[string]string{
    "a": "A",
    "b": "B",
}

mv := reflect.ValueOf(m)
for _, k := range mv.MapKeys() {
    v := mv.MapIndex(k)
    fmt.Printf("%s - %s\n", k, v)
}

// output:
// a - A
// b - B
```

其他类型也有对应的api，具体就查doc吧。