

# 高级特性

## reflect

标准库 [reflect](https://golang.org/pkg/reflect/) 为 Go 语言提供了运行时动态获取对象的类型和值以及动态创建对象的能力。反射可以帮助抽象和简化代码，提高开发效率。


### 重要函数

- [`reflect.TypeOf`](https://draveness.me/golang/tree/reflect.TypeOf) 能获取类型信息；
- [`reflect.ValueOf`](https://draveness.me/golang/tree/reflect.ValueOf) 能获取数据的运行时表示；

两个类型是 [`reflect.Type`](https://draveness.me/golang/tree/reflect.Type) 和 [`reflect.Value`](https://draveness.me/golang/tree/reflect.Value)，它们与函数是一一对应的关系：

![[Pasted image 20240607010323.png]]


类型 [`reflect.Type`](https://draveness.me/golang/tree/reflect.Type) 是反射包定义的一个接口，我们可以使用 [`reflect.TypeOf`](https://draveness.me/golang/tree/reflect.TypeOf) 函数获取任意变量的类型，[`reflect.Type`](https://draveness.me/golang/tree/reflect.Type) 接口中定义了一些有趣的方法，`MethodByName` 可以获取当前类型对应方法的引用、`Implements` 可以判断当前类型是否实现了某个接口：

```go
type Type interface {
        Align() int
        FieldAlign() int
        Method(int) Method
        MethodByName(string) (Method, bool)
        NumMethod() int
        ...
        Implements(u Type) bool
        ...
}
```

反射包中 [`reflect.Value`](https://draveness.me/golang/tree/reflect.Value) 的类型与 [`reflect.Type`](https://draveness.me/golang/tree/reflect.Type) 不同，它被声明成了结构体。这个结构体没有对外暴露的字段，但是提供了获取或者写入数据的方法：

```go
type Value struct {
        // 包含过滤的或者未导出的字段
}

func (v Value) Addr() Value
func (v Value) Bool() bool
func (v Value) Bytes() []byte
...
```


反射包中的所有方法基本都是围绕着 [`reflect.Type`](https://draveness.me/golang/tree/reflect.Type) 和 [`reflect.Value`](https://draveness.me/golang/tree/reflect.Value) 两个类型设计的。我们通过 [`reflect.TypeOf`](https://draveness.me/golang/tree/reflect.TypeOf)、[`reflect.ValueOf`](https://draveness.me/golang/tree/reflect.ValueOf) 可以将一个普通的变量转换成反射包中提供的 [`reflect.Type`](https://draveness.me/golang/tree/reflect.Type) 和 [`reflect.Value`](https://draveness.me/golang/tree/reflect.Value)，随后就可以使用反射包中的方法对它们进行复杂的操作。



### 实践：使用reflect把环境变量读取到结构体当中

通过获取到结构体的标签，来去比对环境变量中是否存在该值，有则赋值。
通过reflect的方式比 swich 或者 if  else的硬编码灵活性更好，后续就算Config结构体有什么变动，也不会影响到已经做好映射的值

```go

type Config struct {  
	Name    string `json:"server-name"` // CONFIG_SERVER_NAME  
	IP      string `json:"server-ip"`   // CONFIG_SERVER_IP  
	URL     string `json:"server-url"`  // CONFIG_SERVER_URL  
	Timeout string `json:"timeout"`     // CONFIG_TIMEOUT  
}

func readConfig() *Config {  
	// read from xxx.json，省略  
	config := Config{}  
	typ := reflect.TypeOf(config)  
	value := reflect.Indirect(reflect.ValueOf(&config))  
	for i := 0; i < typ.NumField(); i++ {  
		f := typ.Field(i)  
		if v, ok := f.Tag.Lookup("json"); ok {  
			key := fmt.Sprintf("CONFIG_%s", strings.ReplaceAll(strings.ToUpper(v), "-", "_"))  
			if env, exist := os.LookupEnv(key); exist {  
				value.FieldByName(f.Name).Set(reflect.ValueOf(env))  
			}  
		}  
	}  
	return &config  
}  
  
func main() {  
	os.Setenv("CONFIG_SERVER_NAME", "global_server")  
	os.Setenv("CONFIG_SERVER_IP", "10.0.0.1")  
	os.Setenv("CONFIG_SERVER_URL", "geektutu.com")  
	c := readConfig()  
	fmt.Printf("%+v", c)  
}
```




# 并发编程



## Context

context用来设置截止日期、同步信号、传递请求相关值的结构体，context作为一个接口，需要实现四个函数
* Deadline: 返回工作完成的截止时间
* Done: 返回一个channel，这个channel会在工作完成或时间截至后返回，多次调用返回同一个channel
* Err：返回对应的错误信息
* Value:从context当中获取健对应的值，多次调用Value并传入相同的Key会返回同样的结果


```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```


使用context对信号进行同步
```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	go handle(ctx, 500*time.Millisecond)
	select {
	case <-ctx.Done():
		fmt.Println("main", ctx.Err())
	}
}

func handle(ctx context.Context, duration time.Duration) {
	select {
	case <-ctx.Done():
		fmt.Println("handle", ctx.Err())
	case <-time.After(duration):
		fmt.Println("process request with", duration)
	}
}
```



#### 默认上下文

`context.Backgroud & context.TODO` 这两个方法返回的对象，都会只想`context.emptyCtx`这个空结构体，
```go
func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```


emptyCtx
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

从源代码来看，[`context.Background`](https://draveness.me/golang/tree/context.Background) 和 [`context.TODO`](https://draveness.me/golang/tree/context.TODO) 也只是互为别名，没有太大的差别，只是在使用和语义上稍有不同：

- [`context.Background`](https://draveness.me/golang/tree/context.Background) 是上下文的默认值，所有其他的上下文都应该从它衍生出来；
- [`context.TODO`](https://draveness.me/golang/tree/context.TODO) 应该仅在不确定应该使用哪种上下文时使用；


#### 取消信号


`context.WithCancel`函数能够从 [`context.Context`](https://draveness.me/golang/tree/context.Context) 中衍生出一个新的子上下文并返回用于取消该上下文的函数。一旦我们执行返回的取消函数，当前上下文以及它的子上下文都会被取消，所有的 Goroutine 都会同步收到这一取消信号。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

- [`context.newCancelCtx`](https://draveness.me/golang/tree/context.newCancelCtx) 将传入的上下文包装成私有结构体 [`context.cancelCtx`](https://draveness.me/golang/tree/context.cancelCtx)；
- [`context.propagateCancel`](https://draveness.me/golang/tree/context.propagateCancel) 会构建父子上下文之间的关联，当父上下文被取消时，子上下文也会被取消：


```go
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // 父上下文不会触发取消信号
	}
	select {
	case <-done:
		child.cancel(false, parent.Err()) // 父上下文已经被取消
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			child.cancel(false, p.err)
		} else {
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

上述函数总共与父上下文相关的三种不同的情况：

1. 当 `parent.Done() == nil`，也就是 `parent` 不会触发取消事件时，当前函数会直接返回；
2. 当 `child` 的继承链包含可以取消的上下文时，会判断 `parent` 是否已经触发了取消信号；
    - 如果已经被取消，`child` 会立刻被取消；
    - 如果没有被取消，`child` 会被加入 `parent` 的 `children` 列表中，等待 `parent` 释放取消信号；
3. 当父上下文是开发者自定义的类型、实现了 [`context.Context`](https://draveness.me/golang/tree/context.Context) 接口并在 `Done()` 方法中返回了非空的管道时；
    1. 运行一个新的 Goroutine 同时监听 `parent.Done()` 和 `child.Done()` 两个 Channel；
    2. 在 `parent.Done()` 关闭时调用 `child.cancel` 取消子上下文；

[`context.propagateCancel`](https://draveness.me/golang/tree/context.propagateCancel) 的作用是在 `parent` 和 `child` 之间同步取消和结束的信号，保证在 `parent` 被取消时，`child` 也会收到对应的信号，不会出现状态不一致的情况。

[`context.cancelCtx`](https://draveness.me/golang/tree/context.cancelCtx) 实现的几个接口方法也没有太多值得分析的地方，该结构体最重要的方法是 [`context.cancelCtx.cancel`](https://draveness.me/golang/tree/context.cancelCtx.cancel)，该方法会关闭上下文中的 Channel 并向所有的子上下文同步取消信号：

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```


#### 传值方法

[`context`](https://github.com/golang/go/tree/master/src/context) 包中的 [`context.WithValue`](https://draveness.me/golang/tree/context.WithValue) 能从父上下文中创建一个子上下文，传值的子上下文使用 [`context.valueCtx`](https://draveness.me/golang/tree/context.valueCtx) 类型：

```go
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```


[`context.valueCtx`](https://draveness.me/golang/tree/context.valueCtx) 结构体会将除了 `Value` 之外的 `Err`、`Deadline` 等方法代理到父上下文中，它只会响应 [`context.valueCtx.Value`](https://draveness.me/golang/tree/context.valueCtx.Value) 方法，该方法的实现也很简单：

```go
type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

如果 [`context.valueCtx`](https://draveness.me/golang/tree/context.valueCtx) 中存储的键值对与 [`context.valueCtx.Value`](https://draveness.me/golang/tree/context.valueCtx.Value) 方法中传入的参数不匹配，就会从父上下文中查找该键对应的值直到某个父上下文中返回 `nil` 或者查找到对应的值。