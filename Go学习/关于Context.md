## 核心概念
context.Context是Go语言中用于管理请求生命周期、传递请求域数据和控制并发的核心接口。其主要功能包括：
- 取消传播：通过树状结构传播取消信号
- 超时控制：设置截止时间(Deadline)和超时(Timeout)
- 值传递：在调用链中安全传递请求域数据
- 并发控制：协调多个goroutine的执行

## 本质
context本质就是四个接口函数
```Go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

## 四个接口函数详解

### 1. Deadline() (deadline time.Time, ok bool)
**功能**：返回context的截止时间，用于超时控制
**参数**：无
**返回值**：
- `deadline`：具体的截止时间点
- `ok`：是否设置了截止时间的布尔值

**详细说明**：
- 当`ok`为`true`时，表示context设置了截止时间，`deadline`是具体的时间点
- 当`ok`为`false`时，表示没有设置截止时间，context不会因为超时而取消
- 超时检查通常在长时间运行的操作中使用，如网络请求、数据库查询等

**使用示例**：
```go
if deadline, ok := ctx.Deadline(); ok {
    if time.Now().After(deadline) {
        return errors.New("context已超时")
    }
}
```

### 2. Done() <-chan struct{}
**功能**：返回一个只读channel，当context被取消时会关闭这个channel
**参数**：无
**返回值**：只读的struct{}类型channel

**详细说明**：
- 这是context最重要的方法，用于监听取消信号
- 当context被取消时（主动取消或超时），这个channel会被关闭
- 通过select语句监听这个channel，实现优雅的取消机制
- 如果context永远不会被取消，Done可能返回nil

**使用示例**：
```go
select {
case <-ctx.Done():
    return ctx.Err() // context被取消，退出
case result := <-workChan:
    return result    // 正常完成工作
}
```

### 3. Err() error
**功能**：返回context被取消的原因
**参数**：无
**返回值**：error类型，说明取消的具体原因

**详细说明**：
- 如果Done()返回的channel尚未关闭，Err返回nil
- 如果Done()返回的channel已关闭，Err返回非nil的error
- 常见的错误类型：
  - `context.Canceled`：context被主动取消
  - `context.DeadlineExceeded`：context因超时被取消

**使用示例**：
```go
if err := ctx.Err(); err != nil {
    if err == context.Canceled {
        log.Println("操作被取消")
    } else if err == context.DeadlineExceeded {
        log.Println("操作超时")
    }
    return err
}
```

### 4. Value(key interface{}) interface{}
**功能**：根据key获取context中存储的值
**参数**：`key`：要查找的键
**返回值**：对应的值，如果不存在则返回nil

**详细说明**：
- 用于在调用链中传递请求域数据，如用户ID、请求ID、认证信息等
- key应该是可比较的类型，通常使用自定义类型避免冲突
- 值的查找是递归的，会沿着context树向上查找
- 应该谨慎使用，避免滥用导致隐式依赖

**使用示例**：
```go
// 定义自定义key类型
type userIDKey struct{}

// 存储值
ctx = context.WithValue(ctx, userIDKey{}, "user123")

// 获取值
if userID, ok := ctx.Value(userIDKey{}).(string); ok {
    log.Printf("当前用户ID: %s", userID)
}
```

## 工程实践应用场景
1. HTTP请求超时控制（服务端）
```go
func apiHandler(w http.ResponseWriter, r *http.Request) {
    // 创建带超时的Context (2秒)
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel() // 确保资源释放
    
    // 将Context传递给下游处理
    result, err := processRequest(ctx)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            w.WriteHeader(http.StatusGatewayTimeout)
            fmt.Fprint(w, "请求超时")
            return
        }
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprint(w, "服务器错误")
        return
    }
    
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(result)
}

func processRequest(ctx context.Context) (Result, error) {
    // 模拟耗时操作
    select {
    case <-time.After(3 * time.Second): // 实际耗时3秒
        return Result{Data: "成功"}, nil
    case <-ctx.Done(): // 监听取消信号
        return Result{}, ctx.Err()
    }
}
```

2. 数据库查询取消（客户端）
```go
func getUserProfile(ctx context.Context, userID string) (*UserProfile, error) {
    // 创建带取消功能的Context
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    // 并发获取用户数据
    var (
        profile *UserProfile
        orders  []Order
        err     error
    )
    
    wg := sync.WaitGroup{}
    wg.Add(2)
    
    // 获取基本信息
    go func() {
        defer wg.Done()
        profile, err = db.GetUser(ctx, userID)
        if err != nil {
            cancel() // 出错时取消其他操作
        }
    }()
    
    // 获取订单信息
    go func() {
        defer wg.Done()
        orders, err = db.GetOrders(ctx, userID)
        if err != nil {
            cancel() // 出错时取消其他操作
        }
    }()
    
    wg.Wait()
    
    if err != nil {
        return nil, err
    }
    
    profile.Orders = orders
    return profile, nil
}

// 数据库查询实现
func (db *Database) GetUser(ctx context.Context, id string) (*User, error) {
    // 检查Context是否已取消
    if err := ctx.Err(); err != nil {
        return nil, err
    }
    
    // 将Context传递给SQL驱动
    row := db.conn.QueryRowContext(ctx, "SELECT ... WHERE id = ?", id)
    
    // 解析结果...
}
```
这里的原理，数据库驱动（如database/sql）在内部会实时监听context的取消信号
```go
// 简化的数据库驱动实现
func (conn *Connection) executeQuery(ctx context.Context, query string) {
    // 启动监听goroutine
    go func() {
        select {
        case <-ctx.Done():
            // 收到取消信号，立即关闭连接
            conn.close()
            return
        case <-conn.queryDone:
            // 查询正常完成
            return
        }
    }()
    
    // 执行实际查询...
}
```
当被context被cancel时，会关闭ctx.Done() 这个channel，信号会传播给所有监听该context的goroutine与所有基于这个context创建的子context。比如数据库驱动代码，在以上的select的监听中完成了关闭数据库连接。

3. 跨服务分布式追踪
```go
// 追踪中间件
func TracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 从请求头提取追踪ID
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = generateTraceID()
        }
        
        // 创建包含追踪ID的Context
        ctx := context.WithValue(r.Context(), traceKey{}, traceID)
        
        // 注入响应头
        w.Header().Set("X-Trace-ID", traceID)
        
        // 传递新Context
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 业务处理函数
func businessHandler(w http.ResponseWriter, r *http.Request) {
    // 从Context获取追踪ID
    ctx := r.Context()
    traceID, ok := ctx.Value(traceKey{}).(string)
    if !ok {
        traceID = "unknown"
    }
    
    // 记录带追踪ID的日志
    log.Printf("[%s] Handling request", traceID)
    
    // 调用下游服务时传递追踪ID
    callDownstreamService(ctx)
}

// 下游服务调用
func callDownstreamService(ctx context.Context) {
    traceID, _ := ctx.Value(traceKey{}).(string)
    
    req, _ := http.NewRequestWithContext(
        ctx, 
        "GET", 
        "https://downstream/api",
        nil,
    )
    req.Header.Set("X-Trace-ID", traceID)
    
    // 发送请求...
}

// 专用key类型防止冲突
type traceKey struct{}
```

4. 优雅关闭服务
```go
func main() {
    // 创建根Context和取消函数
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // 启动HTTP服务器
    server := &http.Server{Addr: ":8080", Handler: router}
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("HTTP server error: %v", err)
        }
    }()
    
    // 启动后台任务
    go runBackgroundTask(ctx)
    
    // 等待中断信号
    signalChan := make(chan os.Signal, 1)
    signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)
    <-signalChan
    
    log.Println("Shutting down server...")
    
    // 创建关闭超时Context
    shutdownCtx, shutdownCancel := context.WithTimeout(
        context.Background(), 
        10*time.Second,
    )
    defer shutdownCancel()
    
    // 优雅关闭HTTP服务器
    if err := server.Shutdown(shutdownCtx); err != nil {
        log.Fatalf("HTTP shutdown error: %v", err)
    }
    
    // 取消所有Context，通知后台任务退出
    cancel()
    
    log.Println("Server stopped gracefully")
}

func runBackgroundTask(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            // 执行定期任务
            processTask()
        case <-ctx.Done():
            // 收到关闭信号
            log.Println("Background task stopping")
            // 执行清理工作...
            return
        }
    }
}
```

5. 级联超时控制
```go
func processOrder(ctx context.Context, order Order) error {
    // 创建带超时的子Context（保留原有值）
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    
    // 级联调用多个服务
    errGroup, gCtx := errgroup.WithContext(ctx)
    
    // 验证库存
    errGroup.Go(func() error {
        return inventoryService.CheckStock(gCtx, order.Items)
    })
    
    // 验证用户
    errGroup.Go(func() error {
        return userService.ValidateUser(gCtx, order.UserID)
    })
    
    // 计算价格
    errGroup.Go(func() error {
        return pricingService.Calculate(gCtx, order)
    })
    
    // 等待所有操作完成
    if err := errGroup.Wait(); err != nil {
        return fmt.Errorf("order processing failed: %w", err)
    }
    
    // 创建支付Context（更短的超时）
    payCtx, payCancel := context.WithTimeout(ctx, 1*time.Second)
    defer payCancel()
    
    // 执行支付
    if err := paymentService.Process(payCtx, order); err != nil {
        return fmt.Errorf("payment failed: %w", err)
    }
    
    return nil
}
```

## 最佳实践
1. 作为函数第一个参数：
```go
// 正确
func Process(ctx context.Context, data Data) error

// 避免
func Process(data Data, ctx context.Context) error
```

2. 避免存储Context在结构体中：
```go
// 不推荐
type Service struct {
    ctx context.Context
}

// 推荐：在方法中传递
func (s *Service) Process(ctx context.Context) error
```

3. 使用专用键类型防止冲突：
```go
type userKey struct{}

func WithUser(ctx context.Context, user *User) context.Context {
    return context.WithValue(ctx, userKey{}, user)
}

func UserFromContext(ctx context.Context) (*User, bool) {
    user, ok := ctx.Value(userKey{}).(*User)
    return user, ok
}
```

4、取消释放资源，用defer
```go
ctx, cancel := context.WithCancel(parentCtx)
defer cancel() // 确保总是调用cancel
```