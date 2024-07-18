

## 框架设计理念

已经有了net标准库来处理网络请求，为什么还需要gin框架，gin框架解决了什么问题


1. 动态路由功能的添加
2. 请求和响应的封装和简化
3. 分组，统一鉴权的能力
4. 中间件，日志信息的处理


## 请求的统一处理


http标准库里面，启动一个服务示例如下:
```go
package main  
  
import (  
	"fmt"  
	"log"  
	"net/http"  
)  
  
func main() {  
	http.HandleFunc("/", indexHandler)  
	http.HandleFunc("/hello", helloHandler)  
	log.Fatal(http.ListenAndServe(":9999", nil))  
}

// handler echoes r.URL.Path  
func indexHandler(w http.ResponseWriter, req *http.Request) {  
	fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)  
}  
  
// handler echoes r.URL.Header  
func helloHandler(w http.ResponseWriter, req *http.Request) {  
	for k, v := range req.Header {  
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)  
	}  
}
```


在这当中，`http.ListenAndServer`接受两个参数，其中第一个为端口号，第二个就是一个Handler，
这个Handler为一个包含ServerHTTP的接口， 所以，只要我们提供一个实现了ServerHTTP方法的结构体，就可以把它当成是Handler接口，ListenAndServer函数当中就会对每一个请求开启一个协程，并且都会调用ServerHTTP方法

```go
// http server
package http  
  
type Handler interface {  
    ServeHTTP(w ResponseWriter, r *Request)  
}  
  
func ListenAndServe(address string, h Handler) error
```


简单的实现如下：
```go
// HandlerFunc defines the request handler used by gee  
type HandlerFunc func(http.ResponseWriter, *http.Request)  
  
// Engine implement the interface of ServeHTTP  
type Engine struct {  
	router map[string]HandlerFunc  
}  
  
// New is the constructor of gee.Engine  
func New() *Engine {  
	return &Engine{router: make(map[string]HandlerFunc)}  
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {  
	key := method + "-" + pattern  
	engine.router[key] = handler  
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {  
	key := req.Method + "-" + req.URL.Path  
	if handler, ok := engine.router[key]; ok {  
		handler(w, req)  
	} else {  
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)  
	}
}
```


## Context上下文

Context上下文伴随着一次请求的出现而出现，消亡而消亡那个，记录了这次请求的所有信息，对请求参数和响应进行了封装，简化了用户的操作难度。

```go
ype H map[string]interface{}  
  
type Context struct {  
	// origin objects  
	Writer http.ResponseWriter  
	Req    *http.Request  
	// request info  
	Path   string  
	Method string  
	// response info  
	StatusCode int  
}  
  
func newContext(w http.ResponseWriter, req *http.Request) *Context {  
	return &Context{  
		Writer: w,  
		Req:    req,  
		Path:   req.URL.Path,  
		Method: req.Method,  
	}  
}  
  
func (c *Context) PostForm(key string) string {  
	return c.Req.FormValue(key)  
}  
  
func (c *Context) Query(key string) string {  
	return c.Req.URL.Query().Get(key)  
}  
  
func (c *Context) Status(code int) {  
	c.StatusCode = code  
	c.Writer.WriteHeader(code)  
}  
  
func (c *Context) SetHeader(key string, value string) {  
	c.Writer.Header().Set(key, value)  
}  
  
func (c *Context) String(code int, format string, values ...interface{}) {  
	c.SetHeader("Content-Type", "text/plain")  
	c.Status(code)  
	c.Writer.Write([]byte(fmt.Sprintf(format, values...)))  
}  
  
func (c *Context) JSON(code int, obj interface{}) {  
	c.SetHeader("Content-Type", "application/json")  
	c.Status(code)  
	encoder := json.NewEncoder(c.Writer)  
	if err := encoder.Encode(obj); err != nil {  
		http.Error(c.Writer, err.Error(), 500)  
	}  
}  
  
func (c *Context) Data(code int, data []byte) {  
	c.Status(code)  
	c.Writer.Write(data)  
}  
  
func (c *Context) HTML(code int, html string) {  
	c.SetHeader("Content-Type", "text/html")  
	c.Status(code)  
	c.Writer.Write([]byte(html))  
}
```



## 前缀树路由
前缀树路由通常是动态路由的一种实现方式，其原理是把不同路由当中的相同前缀提取出来，并对其中的特殊符号进行特殊处理

前缀树示例如下：
![[Pasted image 20240711122748.png]]

HTTP请求的路径恰好是由`/`分隔的多段构成的，因此，每一段可以作为前缀树的一个节点。我们通过树结构查询，如果中间某一层的节点都不满足条件，那么就说明没有匹配到的路由，查询结束。

代码实现：
```go

type node struct {  
	pattern  string // 待匹配路由，例如 /p/:lang  
	part     string // 路由中的一部分，例如 :lang  
	children []*node // 子节点，例如 [doc, tutorial, intro]  
	isWild   bool // 是否精确匹配，part 含有 : 或 * 时为true  
}

// 第一个匹配成功的节点，用于插入  
func (n *node) matchChild(part string) *node {  
	for _, child := range n.children {  
		if child.part == part || child.isWild {  
			return child  
		}  
	}  
	return nil  
}  

// 所有匹配成功的节点，用于查找  
func (n *node) matchChildren(part string) []*node {  
	nodes := make([]*node, 0)  
	for _, child := range n.children {  
		if child.part == part || child.isWild {  
			nodes = append(nodes, child)  
		}  
	}  
	return nodes  
}

func (n *node) insert(pattern string, parts []string, height int) {  
	if len(parts) == height {  
		n.pattern = pattern  
		return  
	}  
  
	part := parts[height]  
	//先找到每个节点是否有匹配的子节点，没有则创建
	child := n.matchChild(part)  
	if child == nil {  
		child = &node{part: part, isWild: part[0] == ':' || part[0] == '*'}  
		n.children = append(n.children, child)  
	}
	//递归进行插入
	child.insert(pattern, parts, height+1)  
}  
  
func (n *node) search(parts []string, height int) *node {  
	if len(parts) == height || strings.HasPrefix(n.part, "*") {  
		if n.pattern == "" {  
			return nil  
		}  
		return n  
	}  
  
	part := parts[height]  
	//遍历所有子节点，bfs进行查找
	children := n.matchChildren(part)  
  
	for _, child := range children {  
		result := child.search(parts, height+1)  
		if result != nil {  
			return result  
		}  
	}  
  
	return nil  
}
```



## 分组控制

我们需要对不同的路由进行不同的分组管理，如/admin进行鉴权，/ 根目录进行日志处理等等，这种场景下我们就会涉及到路由的分组功能。对于不同的路由应用不同的中间件进行管理

gee的实现方式是通过实现一个`RouterGroup`，随后`Engine`作为最顶层，把这个`RouterGroup`的功能也组合进来，从而实现路由的分组


```go

RouterGroup struct {  
	prefix      string  
	middlewares []HandlerFunc // support middleware  
	parent      *RouterGroup  // support nesting  
	engine      *Engine       // all groups share a Engine instance  
}
Engine struct {  
	*RouterGroup  
	router *router  
	groups []*RouterGroup // store all groups  
}


//  路由的功能都交给routerGroup来实现了
func New() *Engine {  
	engine := &Engine{router: newRouter()}  
	engine.RouterGroup = &RouterGroup{engine: engine}  
	engine.groups = []*RouterGroup{engine.RouterGroup}  
	return engine  
}  
  
// Group is defined to create a new RouterGroup  
// remember all groups share the same Engine instance  
func (group *RouterGroup) Group(prefix string) *RouterGroup {  
	engine := group.engine  
	newGroup := &RouterGroup{  
		prefix: group.prefix + prefix,  
		parent: group,  
		engine: engine,  
	}  
	engine.groups = append(engine.groups, newGroup)  
	return newGroup  
}  
  
func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {  
	pattern := group.prefix + comp  
	log.Printf("Route %4s - %s", method, pattern)  
	group.engine.router.addRoute(method, pattern, handler)  
}  
  
// GET defines the method to add GET request  
func (group *RouterGroup) GET(pattern string, handler HandlerFunc) {  
	group.addRoute("GET", pattern, handler)  
}  
  
// POST defines the method to add POST request  
func (group *RouterGroup) POST(pattern string, handler HandlerFunc) {  
	group.addRoute("POST", pattern, handler)  
}
```


## 错误处理

gin当中也使用了recover来对错误信息进行统一收集，并防止程序崩溃掉

```go
package gee  
  
import (  
	"fmt"  
	"log"  
	"net/http"  
	"runtime"  
	"strings"  
)  
  
// print stack trace for debug  
func trace(message string) string {  
	var pcs [32]uintptr  
	n := runtime.Callers(3, pcs[:]) // skip first 3 caller  
  
	var str strings.Builder  
	str.WriteString(message + "\nTraceback:")  
	for _, pc := range pcs[:n] {  
		fn := runtime.FuncForPC(pc)  
		file, line := fn.FileLine(pc)  
		str.WriteString(fmt.Sprintf("\n\t%s:%d", file, line))  
	}  
	return str.String()  
}  
  
func Recovery() HandlerFunc {  
	return func(c *Context) {  
		defer func() {  
			if err := recover(); err != nil {  
				message := fmt.Sprintf("%s", err)  
				log.Printf("%s\n\n", trace(message))  
				c.Fail(http.StatusInternalServerError, "Internal Server Error")  
			}  
		}()  
  
		c.Next()  
	}  
}
```