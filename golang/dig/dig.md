[TOC]

### Dig

#### Container

 Dig将类型容器公开为能够解析有向非循环依赖关系图的对象。使用`new`函数创建容器实例

```go
c := dig.New()
```

#### Provide

通过使用`Provide`方法，可以将不同类型的构造函数添加到`container`容器中。构造函数可以简单的将另一个类型作为函数的参数添加来声明对它的依赖关系。类型的依赖项可以在添加类型之前和之后添加到无环类型图中。

```go
err := c.Provide(func(conn *sql.DB) (*UserGateway, error) {
  // ...
})
if err != nil {
  // ...
}

if err := c.Provide(newDBConnection); err != nil {
  // ...
}
```

 多个构造函数可以依赖于同一类型。容器为每个保留类型创建一个单例，当直接请求或作为另一个类型的依赖项时，最多实例化一次。 

```go
err := c.Provide(func(conn *sql.DB) *CommentGateway {
  // ...
})
if err != nil {
  // ...
}
```

构造函数可以将任意数量的依赖项声明为参数，也可以选择返回错误

```go
err := c.Provide(func(u *UserGateway, c *CommentGateway) (*RequestHandler, error) {
  // ...
})
if err != nil {
  // ...
}

if err := c.Provide(newHTTPServer); err != nil {
  // ...
}
```

构造函数也可以返回多个结果来向容器中添加多个类型

```go
err := c.Provide(func(conn *sql.DB) (*UserGateway, *CommentGateway, error) {
  // ...
})
if err != nil {
  // ...
}
```

 接受可变数量参数的构造函数被视为没有这些参数 ,`options`可以不传任何值

```go
func NewVoteGateway(db *sql.DB, options ...Option) *VoteGateway
```

eg:

```go
func NewVoteGateway(db *sql.DB) *VoteGateway
```

上面这个构造函数将与所有其他依赖项一起调用，并且没有可变参数。

#### InVoke

可以使用`Invoke`方法使用添加到容器中的类型。`Invoke`接受一个或多个参数的任何函数，并可选的返回一个`error`。`Dig`使用所请求的类型调用函数，仅实例化该函数所请求的类型。如果容器中没有任何类型或其他依赖项（直接的和传递的），则调用失败。

```go
err := c.Invoke(func(l *log.Logger) {
  // ...
})
if err != nil {
  // ...
}

err := c.Invoke(func(server *http.Server) error {
  // ...
})
if err != nil {
  // ...
}
```

被调用的函数返回的任何错误都会传回给调用者。

#### Parameter Objects

 构造函数将它们的依赖关系声明为函数参数。如果构造函数有很多依赖项，这很快就会变得不可读。 

```go
func NewHandler(users *UserGateway, comments *CommentGateway, posts *PostGateway, votes *VoteGateway, authz *AuthZGateway) *Handler {
  // ...
}
```

 在这种情况下，用来提高可读性的一种模式是创建一个结构，该结构将函数的所有参数都作为字段列出，并更改函数以接受该结构。这称为参数对象。 

 `Dig`对参数对象有第一个类支持:任何结构嵌入`Dig.In`被当作一个参数对象。下面的代码等价于上面的构造函数。 

```go
type HandlerParams struct {
  dig.In

  Users    *UserGateway
  Comments *CommentGateway
  Posts    *PostGateway
  Votes    *VoteGateway
  AuthZ    *AuthZGateway
}

func NewHandler(p HandlerParams) *Handler {
  // ...
}
```

`Handlers`可以接收参数对象和参数的任何组合

```go
func NewHandler(p HandlerParams, l *log.Logger) *Handler {
  // ...
}
```



#### Result Objects

`Result Objects`对象是`Parameter Objects`的反面，这些结构体将单个函数的多个输出表示为结构体中的字段。结构体中嵌入`dig.out`被当做结果对象

``` go
func SetupGateways(conn *sql.DB) (*UserGateway, *CommentGateway, *PostGateway, error) {
  // ...
}
```

上面的等价如下形式

```go
type Gateways struct {
  dig.Out

  Users    *UserGateway
  Comments *CommentGateway
  Posts    *PostGateway
}

func SetupGateways(conn *sql.DB) (Gateways, error) {
  // ...
}
```



#### Optional Dependencies

构造函数通常对某些类型没有严格的依赖关系，并且在没有依赖关系时能够以降级状态进行操作。通过向Dig的字段添加“optional:‘true’”标记 ,`Dig`支持将依赖项声明为可选。在`Dig.In`结构体在具有'`optional:"true"`标记的结构中，`Dig`将其视为可选的。

```go
type UserGatewayParams struct {
  dig.In

  Conn  *sql.DB
  Cache *redis.Client `optional:"true"`
}
```

 如果容器中没有可用的可选字段，构造函数将接收该字段的零值。 

```go
func NewUserGateway(p UserGatewayParams, log *log.Logger) (*UserGateway, error) {
  if p.Cache != nil {
    log.Print("Logging disabled")
  }
  // ...
}
```

将依赖项声明为可选的构造函数必须处理这些依赖项不存在的情况。

可选标记还允许在不破坏构造函数现有使用者的情况下添加新的依赖项。

#### Named Values   

一些用例调用同一类型的的多个值。`Dig`可以使用命名值`Named Values`的方式向容器添加相同类型的多个值。

命名值（Named Values）可以在依赖构造器即`Porvide`的时候通过传递`dig.Name`来生成。这样这个依赖构造器生成的所有只都具有给定的名称。

给定下面的依赖构造器

```go
func NewReadOnlyConnection(...) (*sql.DB, error)
func NewReadWriteConnection(...) (*sql.DB, error)
```

将`sql.DB`通过`dig.Name`使用不同的名字注入到容器中

```go
c.Provide(NewReadOnlyConnection, dig.Name("ro"))
c.Provide(NewReadWriteConnection, dig.Name("rw"))
```

或者，你可以生成一个`dig.Out`结构体，并用`name:".."`来指定对应值得名称并添加到类型图中。

```go
type GatewayParams struct {
  dig.In

  WriteToConn  *sql.DB `name:"rw"`
  ReadFromConn *sql.DB `name:"ro"`
}
```

`name`标签可以与`optional`标签结合使用，以什么依赖是可选的。

```go
type GatewayParams struct {
  dig.In

  WriteToConn  *sql.DB `name:"rw"`
  ReadFromConn *sql.DB `name:"ro" optional:"true"`
}

func NewCommentGateway(p GatewayParams, log *log.Logger) (*CommentGateway, error) {
  if p.ReadFromConn == nil {
    log.Print("Warning: Using RW connection for reads")
    p.ReadFromConn = p.WriteToConn
  }
  // ...
}
```

#### Value Groups

在`Dig 1.2`中加入

`Dig`提供的`value groups`允许消费和生成许多相同类型的值。`Value Groups`允许构造函数将多个值发送到容器中一个指定的无序集合。其他构造函数可以将此集合中的所有值的所有值当一个切片请求。

依赖构造器可以通过`dig.Out`结构体将值发送到值组`value groups`，以标签`group:".."`的形式。

```go
type HandlerResult struct {
  dig.Out

  Handler Handler `group:"server"`
}

func NewHelloHandler() HandlerResult {
  ..
}

func NewEchoHandler() HandlerResult {
  ..
}
```

任意数量的依赖构造器可能会给命名集合提供值。其他一些依赖构造器会请求该集合的所有值通过标记为`group:".."`的切片。这会执行未指定顺序向该组提供值的所有构造函数。

```go
type ServerParams struct {
  dig.In

  Handlers []Handler `group:"server"`
}

func NewServer(p ServerParams) *Server {
  server := newServer()
  for _, h := range p.Handlers {
    server.Register(h)
  }
  return server
}
```

注意值在值组里是无序的。`Dig`不能保证这些值是在生成时是有序的。

Example

```go
type Config struct {
    Prefix string
}

c := dig.New()

// Provide a Config object. This can fail to decode.
err := c.Provide(func() (*Config, error) {
    // In a real program, the configuration will probably be read from a
    // file.
    var cfg Config
    err := json.Unmarshal([]byte(`{"prefix": "[foo] "}`), &cfg)
    return &cfg, err
})
if err != nil {
    panic(err)
}

// Provide a way to build the logger based on the configuration.
err = c.Provide(func(cfg *Config) *log.Logger {
    return log.New(os.Stdout, cfg.Prefix, 0)
})
if err != nil {
    panic(err)
}

// Invoke a function that requires the logger, which in turn builds the
// Config first.
err = c.Invoke(func(l *log.Logger) {
    l.Print("You've been invoked")
})
if err != nil {
    panic(err)
}
```

Output:

```
[foo] You've been invoked
```

