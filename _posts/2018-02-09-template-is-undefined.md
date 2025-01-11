---
layout: post
title: "template is undefined"
categories: programming
last_modified_at: 2018-02-10T17:48:42+08:00
---

> 初学 Go 语言，跟着[Writing Web Applications](https://golang.org/doc/articles/wiki/)里的教程走了一遍，然后自己练习使用`html/template`时却遇到错误：`html/template: *** is undefined`，一个小问题纠结了半个小时。  

<!-- more -->

我的目录结构：  
```
├── main.go
└── views
    └── index.tmpl
```
main.go 代码：  
```go
package main

import (
    "html/template"
    "net/http"
)

var templates = template.Must(template.ParseFiles("views/index.tmpl"))

func indexHandler(w http.ResponseWriter, r *http.Request) {
    err := templates.ExecuteTemplate(w, "views/index.tmpl", nil)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}

func main() {
    http.HandleFunc("/", indexHandler)
    http.ListenAndServe(":8080", nil)
}
```
当我访问[http://localhost:8080/](http://localhost:8080/)时，就会看到报错：`html/template: "views/index.tmpl" is undefined`。  

后来 Google 了一些其他人关于`html/template`的代码，发现自己是在`err := templates.ExecuteTemplate(w, "views/index.tmpl", nil)`一行出现了问题，模仿其他人的代码，把这里的`views/index.tmpl`换成`index.tmpl`后程序就正常了。  

再回头看[Writing Web Applications](https://golang.org/doc/articles/wiki/)的 Template caching 部分，里面对`var templates = template.Must(template.ParseFiles("edit.html", "view.html"))`的解释是有这一点的：  
> The `ParseFiles` function takes any number of string arguments that identify our template files, and parses those files into templates that are named after the base file name. If we were to add more templates to our program, we would add their names to the `ParseFiles` call's arguments.  

大意：`ParseFiles`函数可以接收任意数量用来标识模板文件的字符串，然后将这些模板文件解析为**以基文件名命名的模板**。如果想要向程序中添加更多的模板文件，把它们的文件名称添加到`ParseFiles`函数的参数中就行。  

也就是说添加模板文件之后，生成的模板是以文件名来命名的，调用的时候通过文件名即可调用对应的模板，例如添加的时候用`views/index.tmpl`指明模板文件的路径，生成的模板就以`index.tmpl`命名。所以`ExecuteTemplate`函数中应该直接使用文件名指定模板，导入的`ParseFiles`需要的才是文件路径。  

这么说，模板文件如果在不同文件夹存在相同文件名的情况，导入的时候就会产生冲突。  

尝试之后发现，相同文件名的模板文件导入的时候并不会有报错，但使用文件名调用的时候会调用到后导入的模板文件，先导入的模板文件貌似被覆盖了。以后使用`html/template`的时候还是注意不要有重名文件比较好。
