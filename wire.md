# Wire: Automated Initialization in Go

> https://github.com/google/wire

Wire is a code generation tool
* automates connecting components using dependency injection. 
*  Dependencies between components are represented in Wire as function parameter
*  wire则是采用代码生成的方式来达到编译时依赖注入 compile-time dependency injection



## Provider & Injector

> provider: a function that can produce a value. These functions are ordinary Go code.
> 
> injector: a function that calls providers in dependency order. With Wire, you write the injector’s signature, then Wire generates the function’s body.


**Provider**

普通的Go函数，可以把它看作是某对象的构造函数，我们通过provider告诉wire该对象的依赖情况

```
// NewUserStore是*UserStore的provider，表明*UserStore依赖于*Config和 *mysql.DB.
func  NewUserStore(cfg *Config, db *mysql.DB) (*UserStore, error) {...} 
```

**injector**

injector是wire生成的函数，我们通过调用injector来获取我们所需的对象或值，injector会按照依赖关系，按顺序调用provider函数

```
// initUserStore是由wire生成的injector
func initUserStore(info ConnectionInfo) (*UserStore, error) {
       // *Config的provider函数
      defaultConfig := NewDefaultConfig()
      // *mysql.DB的provider函数
      db, err := NewDB(info)
      // *UserStore的provider函数
      userStore, err := NewUserStore(defaultConfig, db)
}
```
injector帮我们把按顺序初始化依赖的步骤给做了，我们在main.go中只需要调用initUserStore方法就能得到我们想要的对象了

**wire是怎么知道如何生成injector的呢？**

* 定义injector的函数签名
* 在函数中使用wire.Build方法列举生成injector所需的provider
```
wire.Build(UserStoreSet, NewDB)
```

wire生成injector的步骤
1. 确定所生成injector函数的函数签名：`func initUserStore(info ConnectionInfo) (*UserStore, error)`
2. 感知返回值第一个参数是`*UserStore`
3. 检查wire.Build列表，找到`*UserStore`的`provider：NewUserStore`
4. 由函数签名`func NewUserStore(cfg *Config, db *mysql.DB)`得知`NewUserStore`依赖于`*Config, 和*mysql.DB`
5. 检查`wire.Build`列表，找到`*Config`和`*mysql.DB`的`provider：NewDefaultConfig`和`NewDB`
6. 由函数签名`func NewDefaultConfig() *Config`得知`*Config`没有其他依赖了。
7. 由函数签名`func NewDB(info *ConnectionInfo) (*mysql.DB, error)`得知`*mysql.D`B依赖于`ConnectionInfo`
8. 检查`wire.Build`列表，找不到`ConnectionInfo`的provider，但在injector函数签名中发现匹配的入参类型，直接使用该参数作为NewDB的入参
9. 按依赖关系，按顺序调用provider函数，拼装injector函数。
`

