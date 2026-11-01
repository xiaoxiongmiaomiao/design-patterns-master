# Go 设计模式落地指南

Go 没有继承与类层次，**大多数 GoF 模式在 Go 中会退化成"小接口 + 组合 + 闭包"**。照搬 Java 写法会产生大量无意义的样板。

## 一、Go 对 GoF 模式的惯用改写

| GoF 模式 | Go 惯用写法 |
| --- | --- |
| 单例 | 包级变量 + `sync.Once` |
| 工厂方法 | `NewXxx(...)` 构造函数 |
| 建造者 | functional options（`type Option func(*Server)`） |
| 抽象工厂 | 通常不需要；用接口 + 组合 |
| 模板方法 | 结构体持有函数字段，或传入回调 |
| 策略 | 小接口 + 多个实现 struct |
| 装饰器 / 中间件 | `func(http.Handler) http.Handler` 链式包装 |
| 代理 | 接口 + 包装 struct |
| 适配器 | 包装 struct 实现目标接口 |
| 责任链 | 中间件切片依次包装 |
| 观察者 | channel + goroutine 广播，或回调切片 |
| 状态 | 接口 + 各状态 struct 实现（无继承） |
| 命令 | 闭包或 `type Command func(ctx) error` |
| 迭代器 | 优先回调/生成器式 `iter`（Go 1.23+ `iter.Seq`） |
| 访问者 | 通常改为接口方法，避免双重分派 |

## 二、核心骨架

### functional options（建造者）

```go
type Server struct {
	addr    string
	timeout time.Duration
	logger  *slog.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
	return func(s *Server) { s.timeout = d }
}

func NewServer(addr string, opts ...Option) *Server {
	s := &Server{addr: addr, timeout: 5 * time.Second, logger: slog.Default()}
	for _, opt := range opts {
		opt(s)
	}
	return s
}
```

必填参数放构造函数，可选参数放 option；option 内做校验并返回 error 时用 `Option func(*Server) error`。

### 策略（小接口 + 注册表）

```go
type Notifier interface {
	Channel() string
	Send(ctx context.Context, userID, content string) error
}

type Registry struct {
	byChannel map[string]Notifier
}

func NewRegistry(items ...Notifier) *Registry {
	r := &Registry{byChannel: make(map[string]Notifier, len(items))}
	for _, it := range items {
		r.byChannel[it.Channel()] = it
	}
	return r
}

func (r *Registry) Get(channel string) (Notifier, bool) {
	n, ok := r.byChannel[channel]
	return n, ok
}
```

接口由**使用方**定义，且尽量小（1~3 个方法）；实现方不要提前声明大接口。

### 中间件链（责任链 + 装饰器）

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}

func WithLogging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		slog.InfoContext(r.Context(), "request", "path", r.URL.Path)
		next.ServeHTTP(w, r)
	})
}
```

### 模板方法（函数字段 / 回调）

```go
type Importer struct {
	Read    func(ctx context.Context, src string) ([]Row, error)
	Validate func(rows []Row) error
	Save    func(ctx context.Context, rows []Row) error
}

func (i *Importer) Run(ctx context.Context, src string) error {
	rows, err := i.Read(ctx, src)
	if err != nil {
		return fmt.Errorf("read: %w", err)
	}
	if err := i.Validate(rows); err != nil {
		return fmt.Errorf("validate: %w", err)
	}
	if err := i.Save(ctx, rows); err != nil {
		return fmt.Errorf("save: %w", err)
	}
	return nil
}
```

### 状态（接口 + 状态 struct）

```go
type OrderState interface {
	Next(ctx context.Context, o *Order) (OrderState, error)
	Name() string
}
```

状态常量用自定义类型 + 常量（`type Status string`），禁止裸 `string`/`int` 散落。

### 单例

```go
var (
	once     sync.Once
	instance *Client
)

func GetClient() *Client {
	once.Do(func() { instance = newClient() })
	return instance
}
```

优先在 `main` 中显式构造并注入，避免隐式全局。

## 三、并发模式

| 模式 | 实现 |
| --- | --- |
| Worker Pool | `jobs chan Job` + N 个 goroutine + `sync.WaitGroup`；限流用 `errgroup.SetLimit` 或带缓冲 channel 信号量 |
| Fan-out / Fan-in | 多 goroutine 写同一 `results chan`，汇聚 goroutine 读取 |
| Pipeline | 阶段间 channel 串联，每阶段 `close` 自己的输出 channel |
| 超时与取消 | `context.WithTimeout` / `WithCancel`，所有阻塞调用接收 ctx |
| 错误聚合 | `golang.org/x/sync/errgroup`（`WithContext` 自动取消） |
| 去重回源 | `golang.org/x/sync/singleflight` |
| 限流 | `golang.org/x/time/rate` 或信号量 channel |
| 免锁共享 | 单 goroutine 持有状态 + channel 通信（"不要通过共享内存通信"） |
| 只执行一次 | `sync.Once` / `sync.OnceFunc`（Go 1.21+） |

要点：每个 goroutine 必须有明确退出条件；channel 由发送方关闭；`select` 必须处理 `ctx.Done()`；禁止在循环中泄漏 goroutine。

## 四、错误处理与接口约定

- 错误即值：函数返回 `error`，用 `fmt.Errorf("...: %w", err)` 包装保留链路。
- 定义哨兵错误（`var ErrNotFound = errors.New(...)`）并用 `errors.Is` 判断。
- 正常业务分支不返回 error；不要把 error 当控制流。
- `panic` 仅限不可恢复的编程错误，禁止跨包传播。
- 结构体字段顺序按语义分组；导出标识符必须有注释（golint 要求）。
- 接口命名：单方法接口用 `-er`（`Reader`、`Notifier`）。
- 组合优于继承：用嵌入（`embedding`）复用，但不要为了复用嵌入大结构体。

## 五、Go 编码规范要点

- `gofmt`（或 `gofumpt`）格式化；`go vet` 必过；`golangci-lint` 作为门禁。
- 包名小写、单数、简短；不用 `common`/`util` 这类无意义包名。
- 导入分组：标准库 / 第三方 / 本地；用 `goimports` 自动整理。
- 接收者命名统一（1~2 字母），同类型不要混用值接收者与指针接收者。
- 测试用表驱动（`tests := []struct{...}`）+ `t.Run` 子测试；并发测试加 `-race`。
- 泛型只在真正消除重复时使用，不要为炫技引入。
- 交付前执行：`gofmt -l .`、`go vet ./...`、`go test -race ./...`。

## 六、Go 模式常见错误

- 照搬 Java 抽象工厂：Go 中用 `NewXxx` + 接口参数即可。
- 接口定义过大（`UserService` 十几个方法）：按使用方拆成小接口。
- 返回具体类型而非接口导致无法替换：构造函数返回类型按"调用方是否需要 mock"决定。
- goroutine 泄漏：无退出条件、channel 无人读、忘记 `cancel()`。
- 中间件顺序写反导致鉴权在日志之后：注意 `Chain` 的逆序包装语义。
- 用 `map` + `sync.Mutex` 到处加锁：改单 goroutine 封闭或 `sync.Map`（读多写少）。
