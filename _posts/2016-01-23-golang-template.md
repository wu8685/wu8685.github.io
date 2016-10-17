---
layout: post
title: Golang Template
categories: [golang]
---

最近刚好有个task是要用Golang把Junit的XML格式report转换成HTML格式，便学习了Golang的template包。

基于template做的那个[tool transforming Junit XML report to HTML](https://github.com/wu8685/htmlizer).

Golang提供了对模板的支持（按照文档的说法，是数据驱动模板，data-driven template），分别在"text/template"和"html/template"两个包下。这两个包在api级别是一致的，只是"html/template"提供了对html文本的更好支持，比如会将一些html中的关键符号（类似'<', '>'之类的）做些转义处理再输出。所以以下就只对"html/template"做下介绍。

# 模板定义 #

### hello world ###

```
t := template.Must(template.New("hello").Parse("hello world"))
t.Execute(os.Stdout, nil)

// output:
// hello
```

此处，*template.Must(\*template.Template, error )*会在*Parse*返回*err*不为*nil*时，调用*panic*。*Must*的引入，说的不好听点，就是为了中和Golang将*error*置于函数返回值这种做法带来的缺点。

### 有了*Must*，我们可以将两句inline在一起。 ###

```
template.Must(template.New("hello").Parse("hello world")).Execute(os.Stdout, nil)
```

### 从文件初始化模板 ###

```
t := template.Must(template.ParseFiles("hello.txt"))
t.Execute(os.Stdout, nil)

// hello.txt:
// hello world

// output:
// hello world
```

### 通过文件名字，指定对应的模板 ###

```
t := template.New("hello")
template.Must(t.ParseFiles("hello.txt"))
template.Must(t.ParseFiles("world.txt"))
t.ExecuteTemplate(os.Stdout, "world.txt", nil)

// world.txt:
// another file

// output:
// another file
```

所以，一个*template*实例，其实是一个模板的集合，我们可以为每个模板命名。

# 数据驱动 #

### hello world ###

{% raw %}
```
s := "LiLei"
t := template.Must(template.New("test").Parse("Watch out, {{.}}!"))
t.Execute(os.Stdout, s)

// output:
// Watch out, LiLei!
```
{% endraw %}

### 也可以传点别的 ###

{% raw %}
```
s := &student{Name: "Han Meimei", Age: 30}
t := template.Must(template.New("test").Parse("{{.Name}} looks like more than {{.Age}} years old!"))
t.Execute(os.Stdout, s)

// output:
// Han Meimei looks like more than 30 years old!
```
{% endraw %}

### Map ###

{% raw %}
```
marrage_info := map[string]bool{
    "HanMeimei": true,
    "LiLei": false,
}
t := template.Must(template.New("test").Parse("Married: Han Meimei:{{.HanMeimei}}; Li Lei:{{.LiLei}}"))
t.Execute(os.Stdout, marrage_info )

// output:
// Married: Han Meimei:true; Li Lei:false
```
{% endraw %}

在template中key有空格，得用到*index*。

{% raw %}
```
info := map[string]bool{
	"Han Meimei": true,
	"LiLei": false,
}
t := template.Must(template.New("test").Parse(`Married: Han Meimei:{{index . "Han Meimei"}}; Li Lei:{{.LiLei}}`))
t.Execute(os.Stdout, info)

// output:
// Married: Han Meimei:true; Li Lei:false
```
{% endraw %}

### array/slice ###

{% raw %}
```
infos := []string{"Han Meimei", "Lilei"}
t := template.Must(template.New("test").Parse("Students List:" +
	"{{range .}}" +
	"\n{{.}}," +
	"{{end}}"))
t.Execute(os.Stdout, infos)

// output:
// Students List:
// Han Meimei,
// Lilei,
```
{% endraw %}

### 自定义变量 ###

{% raw %}
```
infos := []string{"Han Meimei", "Lilei"}
t := template.Must(template.New("test").Parse("Students List:" +
	"{{range $index, $_ := .}}" +
	"\n{{$index}}. {{.}}," +
	"{{end}}"))
t.Execute(os.Stdout, infos)

// output:
// Students List:
// 0. Han Meimei,
// 1. Lilei,
```
{% endraw %}

### 条件查询 ###

{% raw %}
```
s := "LiLei"
t := template.Must(template.New("test").Parse(`{{if eq . "LiLei"}}Man{{else if eq . "Han Meimei"}}Women{{end}}`))
t.Execute(os.Stdout, s)

// output:
// Man
```
{% endraw %}

### 转义 ###

在"html/template"中，如下两端代码是等价的。

{% raw %}
```
<a href="/search?q={{.}}">{{.}}</a>
```
{% endraw %}

{% raw %}
```
<a href="/search?q={{. | urlquery}}">{{. | html}}</a>
```
{% endraw %}

比如

{% raw %}
```
s := "<script>"
t := template.Must(template.New("test").Parse("{{. | urlquery}} and {{. | html}}"))
t.Execute(os.Stdout, s)

// output:
// %3Cscript%3E and <script>
```
{% endraw %}

# 模板重构 #

Golang提供的几个功能，为模板的重构提供了更多的可能。

### 自定义方法 ###

{% raw %}
```
// func returnBool(b bool) bool {
//     return b
// }

funcs := template.FuncMap{
	"is_teacher_coming": returnBool,
}
t := template.New("test").Funcs(funcs)
template.Must(t.Parse("{{if is_teacher_coming .}}Carefully!{{end}}"))
t.Execute(os.Stdout, true)

// output:
// Carefully!
```
{% endraw %}

### 嵌套模板 ###

{% raw %}
```
s := &student{Name: "Han Meimei", Age: 30}
t := template.New("test")
template.Must(t.Parse(`Name: {{.Name}}; {{template "age" .Age}}.`))
// another way to name a template
template.Must(t.Parse(`{{define "age"}}Age: {{.}}{{end}}`))
t.Execute(os.Stdout, s)

// output:
// Name: Han Meimei; Age: 30.
```
{% endraw %}

这里也提供了另外一个方法去指定一个模板的名字。