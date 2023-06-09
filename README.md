这是一个基于go net/http构建的最简单的web服务器demo。翻译了官方教程：https://go.dev/doc/articles/wiki/，并上传了完整项目代码


<a name="SglNT"></a>
# 官方文档
[https://go.dev/doc/articles/wiki/](https://go.dev/doc/articles/wiki/)
<a name="OqE8J"></a>
# 快速开始
<a name="AVFgI"></a>
## 1、最简单的服务器（两步完成服务器搭建）
第一步：设置路径处理器<br />第二步：启动服务器，监听端口号
```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	// 1、设置对应路径的处理器
	http.HandleFunc("/", handler)
	// 2、启动服务器
	http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}
```
<a name="jRizk"></a>
## 2、多handler，读取文件返回html页面
<a name="vODrG"></a>
### 2.1 代码
```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

type Page struct {
	Title string
	Body  []byte
}

func main() {
	// 1、设置对应路径的处理器
	http.HandleFunc("/", handler)
	http.HandleFunc("/view/", viewHandler)
	// 2、启动服务器
	http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

// 读取文件返回html页面
func viewHandler(w http.ResponseWriter, r *http.Request) {
	title := r.URL.Path[len("/view/"):]
	p, _ := loadPage(title)
	fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)
}

func loadPage(title string) (*Page, error) {
	filename := title + ".txt"
	body, err := os.ReadFile(filename)
	if err != nil {
		return nil, err
	}
	return &Page{Title: title, Body: body}, nil
}

```
<a name="lGHxE"></a>
### 2.2 效果
浏览器输入[http://localhost:8080/view/TestPage](http://localhost:8080/view/TestPage)，（本地建好TestPage.txt文件）![image.png](https://cdn.nlark.com/yuque/0/2023/png/12382373/1679045093199-82ceb09c-8973-4cb2-bdf4-fd9f2f05dc3f.png#averageHue=%23ddca31&clientId=ud44073e0-403c-4&from=paste&height=208&id=u2bb4eb39&name=image.png&originHeight=208&originWidth=719&originalType=binary&ratio=2&rotation=0&showTitle=false&size=27019&status=done&style=none&taskId=u9ce2172b-9c39-4f8a-bc78-623ab98875e&title=&width=719)
<a name="X9q9Q"></a>
## 3、使用html/template 包
上面demo中可以看到，viewHandler方法中通过fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)向浏览器返回html标签内容，将前端代码和go代码维护在一个文件里是个灾难，和其他语言一样，go的html/template包提供了能够解析html模版文件的能力，方便独立维护。
<a name="uGfir"></a>
### 3.1 首先创建一个html文件
```html
<h1>Editing {{.Title}}</h1>

<form action="/save/{{.Title}}" method="POST">
  <div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
  <div><input type="submit" value="Save"></div>
</form>
```
<a name="TgJX9"></a>
### 3.2 使用html/template进行解析html文件
解析文件，并将Page对象填充到html中输出给浏览器
```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    // 解析html文件
    t, _ := template.ParseFiles("web/edit.html")
    // 将Page对象填充到html中输出给浏览器
    t.Execute(w, p)
}
```
<a name="cCHtP"></a>
### 3.3 完整的demo
```go
package main

import (
	"fmt"
	"html/template"
	"net/http"
	"os"
)

type Page struct {
	Title string
	Body  []byte
}

func main() {
	// 1、设置对应路径的处理器
	http.HandleFunc("/", handler)
	http.HandleFunc("/edit/", editHandler)
	http.HandleFunc("/save/", saveHandler)
	http.HandleFunc("/view/", viewHandler)
	// 2、启动服务器
	http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func viewHandler(w http.ResponseWriter, r *http.Request) {
	title := r.URL.Path[len("/view/"):]
	p, _ := loadPage(title)
	fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)
}

func editHandler(w http.ResponseWriter, r *http.Request) {
	title := r.URL.Path[len("/edit/"):]
	p, err := loadPage(title)
	if err != nil {
		p = &Page{Title: title}
	}
	t, _ := template.ParseFiles("web/edit.html")
	t.Execute(w, p)
}

func saveHandler(w http.ResponseWriter, r *http.Request) {
	title := r.URL.Path[len("/save/"):]
	body := r.FormValue("body")
	p := &Page{Title: title, Body: []byte(body)}
	p.save()
    // 重定向
	http.Redirect(w, r, "/view/"+title, http.StatusFound)
}

func loadPage(title string) (*Page, error) {
	filename := title + ".txt"
	body, err := os.ReadFile(filename)
	if err != nil {
		return nil, err
	}
	return &Page{Title: title, Body: body}, nil
}

func (p *Page) save() error {
	filename := p.Title + ".txt"
	return os.WriteFile(filename, p.Body, 0600)
}

```
<a name="ZQwvC"></a>
### 3.4 效果
浏览器输入：[http://localhost:8080/edit/TestPage](http://localhost:8080/edit/TestPage)<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/12382373/1679063950158-7a65b54f-258a-4e48-9a14-45185cfca544.png#averageHue=%23fbfbfb&clientId=ud44073e0-403c-4&from=paste&height=448&id=u7a2d3944&name=image.png&originHeight=896&originWidth=1584&originalType=binary&ratio=2&rotation=0&showTitle=false&size=49611&status=done&style=none&taskId=u2a215ef2-1593-414a-8ed2-d7f7945bec0&title=&width=792)

<a name="ytsQn"></a>
## 4、http错误处理
使用： http.Error(w, err.Error(), http.StatusInternalServerError)
```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    t, err := template.ParseFiles(tmpl + ".html")
    if err != nil {
       http.Error(w, err.Error(), http.StatusInternalServerError)
       return
   }
    err = t.Execute(w, p)
    if err != nil {
       http.Error(w, err.Error(), http.StatusInternalServerError)
   }
}
```
<a name="EdXOC"></a>
## 5、template缓存
上面renderTemplate方法中这段代码template.ParseFiles(tmpl + ".html")，每次都需要解析文件，实际生产中这样使用效率会很低效。可以考虑在程序初始化时一次性解析好，后面直接使用：
<a name="n7B9G"></a>
### 5.1 初始化所有文件模版
```go
var templates *template.Template

func init() {
	templates = template.Must(template.ParseFiles("web/edit.html", "web/view.html"))
}
```
<a name="z2kNw"></a>
### �5.2 使用模版
err := templates.ExecuteTemplate(w, tmpl, p)
```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
	err := templates.ExecuteTemplate(w, tmpl, p)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```
<a name="OLBFi"></a>
## 6、安全验证
正如您可能已经观察到的，这个程序有一个严重的安全缺陷：用户可以在服务器上提供要读/写的任意路径。为了缓解这种情况，我们可以编写一个函数，用正则表达式验证标题。<br />使用正则表达式解析路径：
<a name="LUC1p"></a>
### 6.1 初始化正则表达式
```go
var validPath = regexp.MustCompile("^/(edit|save|view)/([a-zA-Z0-9]+)$")
```
<a name="f7sa9"></a>
### 6.2 解析路径
```go
func getTitle(w http.ResponseWriter, r *http.Request) (string, error) {
    // 解析路径
	m := validPath.FindStringSubmatch(r.URL.Path)
	if m == nil {
		http.NotFound(w, r)
		return "", errors.New("invalid Page Title")
	}
	return m[2], nil // The title is the second subexpression.
}

```
<a name="VSid6"></a>
### 6.3 调用getTitle
```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}

func editHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}

func saveHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err = p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```
<a name="DY8cR"></a>
## 7、使用Function Literals(函数字面量) + Closures（闭包）优化
<a name="s7YxD"></a>
### 7.1 问题
从6.3节代码中可以看到，viewHandler，editHandler，saveHandler中有针对异常处理的重复代码。我们可以考虑将这些重复代码进行包装，统一处理。
<a name="SKhxe"></a>
### 7.2 Function Literals(函数字面量)
<a name="X27LY"></a>
#### 7.2.1 字面量
字面量指那种直接代表某个常数值的一种表达形式，如 var b = true、false；int a = 1、2、3等。
<a name="xE0g3"></a>
#### 7.2.2 函数字面量
就是将函数作为某个值的表达形式：FunctionLit = "func" Signature FunctionBody <br />如：f := func(x, y int) int { return x + y }<br />func(x, y int) int { return x + y }就是一个具体的函数字面量
<a name="lYoet"></a>
### 7.3 闭包
当一个函数在方法体中使用了函数外部的变量时，称这个函数为闭包。
<a name="WkCOD"></a>
### 7.4 优化
正如7.1中描述的问题，我们尝试这样一个优化思路：<br />定义一个函数makeHandler，在该函数中返回一个函数字面量，该函数字面量满足http.HandleFunc的handler func(ResponseWriter, *Request)��参数，将viewHandler，editHandler，saveHandler三个方法中重复的代码在函数字面量中实现，然后将viewHandler，editHandler，saveHandler方法作为参数在该函数字面量中进行调用。
```go
func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
	// 返回一个闭包（函数字面量）,且该函数作满足http.HandleFunc(pattern string, handler func(ResponseWriter, *Request))的第二个参数类型
	return func(w http.ResponseWriter, r *http.Request) {
        // 重复代码部分start
		m := validPath.FindStringSubmatch(r.URL.Path)
		if m == nil {
			http.NotFound(w, r)
			return
		}
        // 重复代码部分end
        
    	// 具体的handler执行
		fn(w, r, m[2])
	}
}
```
在注册http的handler时，调用makeHandler函数进行注册。如下：
```go
http.HandleFunc("/edit/", makeHandler(editHandler))
http.HandleFunc("/save/", makeHandler(saveHandler))
http.HandleFunc("/view/", makeHandler(viewHandler))
```
<a name="sBteu"></a>
## 8、完成demo链接
[https://github.com/onepiece-dz/go-nethttp-web](https://github.com/onepiece-dz/go-nethttp-web)


