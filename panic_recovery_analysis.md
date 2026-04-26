# Chi 框架 Panic 恢复与错误传播机制分析

## 1. 概述

在 Web 应用中，Panic 是一种严重的运行时错误，如果不加以处理，会导致整个程序崩溃。Chi 框架提供了 `middleware.Recoverer` 中间件来捕获和处理 Panic，确保服务的稳定性。

本文档深入分析以下内容：
- `middleware.Recoverer` 的工作原理
- Panic 发生在中间件链不同位置时的捕获行为
- Panic 恢复后的响应路径
- 与 `http.Error()` 的本质差异
- 最佳实践建议

## 2. Recoverer 中间件深度分析

### 2.1 核心实现

```go
func Recoverer(next http.Handler) http.Handler {
    fn := func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rvr := recover(); rvr != nil {
                if rvr == http.ErrAbortHandler {
                    // 不恢复 http.ErrAbortHandler，让客户端连接被中止
                    panic(rvr)
                }

                logEntry := GetLogEntry(r)
                if logEntry != nil {
                    logEntry.Panic(rvr, debug.Stack())
                } else {
                    PrintPrettyStack(rvr)
                }

                if r.Header.Get("Connection") != "Upgrade" {
                    w.WriteHeader(http.StatusInternalServerError)
                }
            }
        }()

        next.ServeHTTP(w, r)
    }

    return http.HandlerFunc(fn)
}
```
**middleware/recoverer.go:22-49**

### 2.2 关键设计要点

1. **Defer + Recover 模式**：
   - 使用 `defer` 确保函数返回前执行恢复逻辑
   - `recover()` 捕获 panic，返回 panic 的值

2. **特殊处理 `http.ErrAbortHandler`**：
   - 如果 panic 的值是 `http.ErrAbortHandler`，则重新 panic
   - 这是 Go 标准库的特殊约定，用于表示应该中止连接

3. **日志记录**：
   - 优先使用 `GetLogEntry(r)` 记录结构化日志
   - 否则使用 `PrintPrettyStack()` 打印美化的堆栈信息

4. **响应处理**：
   - 非 WebSocket 升级请求，写入 500 状态码
   - WebSocket 升级请求不写入状态码（因为连接已经升级）

### 2.3 美化堆栈打印

```go
func PrintPrettyStack(rvr interface{}) {
    debugStack := debug.Stack()
    s := prettyStack{}
    out, err := s.parse(debugStack, rvr)
    if err == nil {
        recovererErrorWriter.Write(out)
    } else {
        // 降级到标准库输出
        os.Stderr.Write(debugStack)
    }
}
```
**middleware/recoverer.go:54-64**

`prettyStack` 结构体负责将原始堆栈信息解析为更易读的格式，包括：
- 彩色输出（开发环境友好）
- 高亮 panic 发生的位置
- 过滤掉 boilerplate 代码

## 3. 中间件链与 Panic 捕获范围

### 3.1 中间件链的组装方式

理解 Panic 捕获范围之前，需要先理解中间件链的组装方式：

```go
func chain(middlewares []func(http.Handler) http.Handler, endpoint http.Handler) http.Handler {
    if len(middlewares) == 0 {
        return endpoint
    }

    // 从最后一个中间件开始包裹
    h := middlewares[len(middlewares)-1](endpoint)
    for i := len(middlewares) - 2; i >= 0; i-- {
        h = middlewares[i](h)
    }

    return h
}
```
**chain.go:36-49**

### 3.2 洋葱模型执行顺序

假设我们有以下中间件注册顺序：

```go
r.Use(middleware.Logger)      // M1
r.Use(middleware.Recoverer)   // M2
r.Use(authMiddleware)         // M3
r.Get("/", handler)            // H
```

组装后的调用链（从外到内）：

```
M1.ServeHTTP(w, r)
    │
    ├── M1 前置逻辑
    │
    └── M2.ServeHTTP(w, r)
            │
            ├── M2 前置逻辑 (defer recover 注册)
            │
            └── M3.ServeHTTP(w, r)
                    │
                    ├── M3 前置逻辑
                    │
                    └── H.ServeHTTP(w, r)  // 最终 handler
                            │
                            └── (返回)
                    │
                    └── M3 后置逻辑
            │
            └── M2 后置逻辑 (defer 执行，检查 panic)
    │
    └── M1 后置逻辑
```

### 3.3 Panic 捕获范围分析

基于上述洋葱模型，分析不同位置的 Panic 能否被 `Recoverer` 捕获：

#### 场景 1：Panic 发生在最终 Handler 中

```go
r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    panic("something went wrong")  // 这里发生 panic
})
```

**能否被捕获？能**

**执行流程**：
1. M1 执行前置逻辑
2. M2 执行前置逻辑，注册 `defer recover`
3. M3 执行前置逻辑
4. Handler 执行，发生 panic
5. panic 向上传播
6. M2 的 `defer` 执行，`recover()` 捕获 panic
7. 记录日志，写入 500 状态码
8. 正常返回，程序继续运行

#### 场景 2：Panic 发生在 Recoverer 之后的中间件中

```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            panic("missing authorization token")  // 这里发生 panic
        }
        next.ServeHTTP(w, r)
    })
}
```

**能否被捕获？能**

**执行流程**：
1. M1 执行前置逻辑
2. M2 执行前置逻辑，注册 `defer recover`
3. M3 执行前置逻辑，发生 panic
4. panic 向上传播
5. M2 的 `defer` 执行，`recover()` 捕获 panic
6. 记录日志，写入 500 状态码

#### 场景 3：Panic 发生在 Recoverer 之前的中间件中

```go
func loggerMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        panic("logger initialization failed")  // 这里发生 panic
        next.ServeHTTP(w, r)
        log.Printf("request took %v", time.Since(start))
    })
}
```

**能否被捕获？不能**

**执行流程**：
1. M1 执行前置逻辑，发生 panic
2. panic 向上传播
3. M2 的 `defer recover` 还没有被注册（因为 M2 还没开始执行）
4. panic 继续向上传播，直到被 Go runtime 捕获
5. **程序崩溃！**

### 3.4 关键结论

| Panic 发生位置 | 能否被 Recoverer 捕获 | 原因 |
|---------------|----------------------|------|
| Recoverer 之前的中间件 | ❌ 不能 | defer recover 还未注册 |
| Recoverer 中间件本身 | ✅ 能（部分） | defer 会捕获自身作用域内的 panic |
| Recoverer 之后的中间件 | ✅ 能 | 在 defer recover 的作用域内 |
| 最终 Handler | ✅ 能 | 在 defer recover 的作用域内 |

**重要**：`Recoverer` 中间件的注册位置至关重要！它应该尽可能早地注册，以便捕获更多的 panic。

## 4. Panic 恢复后的响应路径

### 4.1 正常响应流程 vs Panic 恢复响应流程

#### 正常响应流程：

```
请求到达
    ↓
M1.Logger 前置逻辑
    ↓
M2.Recoverer 前置逻辑 (注册 defer)
    ↓
M3.Auth 前置逻辑
    ↓
Handler 执行业务逻辑
    ↓
Handler 调用 w.WriteHeader(200) + w.Write(body)
    ↓
M3.Auth 后置逻辑
    ↓
M2.Recoverer 后置逻辑 (defer 执行，无 panic)
    ↓
M1.Logger 后置逻辑
    ↓
响应返回给客户端
```

#### Panic 恢复响应流程：

```
请求到达
    ↓
M1.Logger 前置逻辑
    ↓
M2.Recoverer 前置逻辑 (注册 defer)
    ↓
M3.Auth 前置逻辑
    ↓
Handler 执行 → 发生 panic("error")
    ↓
panic 向上传播，跳过后续代码
    ↓
M2.Recoverer 的 defer 执行
    ├── recover() 捕获 panic
    ├── 记录日志和堆栈信息
    └── 调用 w.WriteHeader(500)
    ↓
M1.Logger 后置逻辑 (如果 M1 没有 panic)
    ↓
响应返回给客户端 (状态码 500)
```

### 4.2 响应写入的关键点

1. **ResponseWriter 的状态**：
   - 如果 panic 发生前已经写入了部分响应，`WriteHeader(500)` 可能无效
   - HTTP 协议规定状态码必须在 body 之前发送

2. **Defer 执行顺序**：
   - defer 按照后进先出（LIFO）的顺序执行
   - `Recoverer` 的 defer 会在其内层所有函数返回后执行

3. **特殊情况：WebSocket 升级**：
   ```go
   if r.Header.Get("Connection") != "Upgrade" {
       w.WriteHeader(http.StatusInternalServerError)
   }
   ```
   - WebSocket 升级请求不写入 500，因为连接已经切换协议

### 4.3 代码层面的响应流程

```go
func Recoverer(next http.Handler) http.Handler {
    fn := func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rvr := recover(); rvr != nil {
                // ... 省略日志记录
                
                // 关键点 1：写入 500 状态码
                if r.Header.Get("Connection") != "Upgrade" {
                    w.WriteHeader(http.StatusInternalServerError)
                }
            }
        }()

        // 关键点 2：调用后续处理器
        next.ServeHTTP(w, r)
    }

    return http.HandlerFunc(fn)
}
```

**关键点分析**：

1. **`defer` 的作用域**：
   - `defer` 捕获的是这个匿名函数作用域内的所有 panic
   - 包括 `next.ServeHTTP()` 调用链中的所有 panic

2. **`recover()` 的返回值**：
   - 如果没有 panic，`recover()` 返回 `nil`
   - 如果有 panic，返回 panic 的值

3. **响应写入时机**：
   - 只有在 `recover()` 捕获到 panic 后才会写入 500
   - 如果 `next.ServeHTTP()` 正常返回，`defer` 中的代码也会执行，但 `rvr` 为 `nil`

## 5. 与 http.Error() 的本质差异

### 5.1 http.Error() 的工作方式

`http.Error()` 是 Go 标准库提供的函数：

```go
func Error(w ResponseWriter, error string, code int) {
    w.Header().Set("Content-Type", "text/plain; charset=utf-8")
    w.Header().Set("X-Content-Type-Options", "nosniff")
    w.WriteHeader(code)
    fmt.Fprintln(w, error)
}
```

### 5.2 核心差异对比

| 维度 | Panic + Recoverer | http.Error() |
|------|-------------------|--------------|
| **性质** | 运行时异常，非预期错误 | 正常的错误处理，预期内的错误 |
| **执行流程** | 中断正常流程，向上传播 | 正常的函数调用，继续执行后续代码 |
| **堆栈信息** | 自动捕获完整堆栈 | 需要手动记录（如果需要） |
| **响应状态码** | 固定 500（可自定义） | 可指定任意状态码（400, 401, 404, 500 等） |
| **响应体** | 默认无响应体（可自定义） | 默认写入错误信息文本 |
| **程序影响** | 不捕获会导致程序崩溃 | 完全不影响程序运行 |
| **使用场景** | 非预期的运行时错误（空指针、数组越界等） | 预期内的业务错误（参数无效、权限不足等） |

### 5.3 代码示例对比

#### 示例 1：使用 http.Error()（推荐用于预期错误）

```go
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "id")
    
    user, err := db.GetUser(userID)
    if err != nil {
        if err == sql.ErrNoRows {
            // 预期内的错误：用户不存在
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }
        // 非预期错误：数据库连接问题
        log.Printf("database error: %v", err)
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    // 正常响应
    json.NewEncoder(w).Encode(user)
}
```

#### 示例 2：Panic 场景（非预期错误）

```go
func riskyHandler(w http.ResponseWriter, r *http.Request) {
    var data *Data
    // 假设这里应该初始化 data，但遗漏了
    
    // 这里会发生 panic：空指针解引用
    fmt.Fprintf(w, "Value: %s", data.Name)
}
```

#### 示例 3：混合使用（最佳实践）

```go
func processHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 预期内的错误：使用 http.Error()
    input := r.FormValue("input")
    if input == "" {
        http.Error(w, "input is required", http.StatusBadRequest)
        return
    }
    
    // 2. 可能发生 panic 的操作
    // Recoverer 会捕获这里的 panic
    result := riskyOperation(input)
    
    // 3. 另一个预期内的错误
    if result == nil {
        http.Error(w, "operation returned no result", http.StatusNotFound)
        return
    }
    
    // 正常响应
    json.NewEncoder(w).Encode(result)
}

func riskyOperation(input string) *Result {
    // 这里可能发生 panic
    // 例如：数组越界、类型断言失败等
    return &Result{Value: input[100]}  // 如果 input 长度不足 100，会 panic
}
```

### 5.4 响应路径差异图示

#### http.Error() 的响应路径：

```
Handler 开始执行
    ↓
执行业务逻辑
    ↓
检测到错误条件
    ↓
调用 http.Error(w, "message", 400)
    ├── 设置 Content-Type 头
    ├── 写入 400 状态码
    └── 写入错误信息到 body
    ↓
调用 return 退出 handler
    ↓
中间件后置逻辑执行
    ↓
响应返回给客户端
```

#### Panic + Recoverer 的响应路径：

```
Handler 开始执行
    ↓
执行业务逻辑
    ↓
发生 panic
    ↓
panic 向上传播，跳过所有后续代码
    ↓
Recoverer 的 defer 执行
    ├── recover() 捕获 panic
    ├── 记录 panic 值和堆栈
    └── 写入 500 状态码
    ↓
外层中间件后置逻辑执行（如果有）
    ↓
响应返回给客户端
```

### 5.5 关键差异总结

1. **控制权**：
   - `http.Error()`：开发者完全控制错误处理流程
   - `Panic`：控制权被转移到 panic 机制，直到被 recover

2. **可预测性**：
   - `http.Error()`：完全可预测，是正常流程的一部分
   - `Panic`：通常不可预测，代表程序 bug 或严重问题

3. **调试信息**：
   - `http.Error()`：需要手动记录上下文和堆栈
   - `Panic`：自动包含完整的堆栈跟踪

4. **状态码灵活性**：
   - `http.Error()`：可以返回任意合适的状态码
   - `Recoverer`：默认返回 500，需要自定义才能返回其他状态码

## 6. 自定义 Recoverer 中间件

### 6.1 为什么需要自定义

默认的 `Recoverer` 存在一些限制：
- 固定返回 500 状态码
- 默认不返回响应体
- 日志格式固定
- 无法根据 panic 类型做不同处理

### 6.2 自定义 Recoverer 示例

```go
func CustomRecoverer(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rvr := recover(); rvr != nil {
                // 1. 特殊处理 http.ErrAbortHandler
                if rvr == http.ErrAbortHandler {
                    panic(rvr)
                }

                // 2. 获取请求 ID（如果有）
                requestID := GetReqID(r.Context())
                
                // 3. 记录详细日志
                log.Printf("[%s] PANIC: %v\nStack:\n%s", 
                    requestID, rvr, debug.Stack())

                // 4. 根据 panic 类型决定响应
                var statusCode int
                var errorMessage string
                
                switch val := rvr.(type) {
                case *MyBusinessError:
                    // 自定义业务错误类型
                    statusCode = val.StatusCode
                    errorMessage = val.Message
                case error:
                    // 标准 error 类型
                    statusCode = http.StatusInternalServerError
                    errorMessage = "Internal server error"
                    // 生产环境不暴露具体错误信息
                    // 开发环境可以：errorMessage = val.Error()
                default:
                    // 其他类型
                    statusCode = http.StatusInternalServerError
                    errorMessage = "Internal server error"
                }

                // 5. 根据请求类型返回不同格式的响应
                contentType := r.Header.Get("Content-Type")
                if strings.Contains(contentType, "application/json") || 
                   strings.Contains(r.Header.Get("Accept"), "application/json") {
                    // JSON 响应
                    w.Header().Set("Content-Type", "application/json")
                    w.WriteHeader(statusCode)
                    json.NewEncoder(w).Encode(map[string]interface{}{
                        "error":       errorMessage,
                        "request_id":  requestID,
                        "status_code": statusCode,
                    })
                } else {
                    // 普通文本响应
                    http.Error(w, errorMessage, statusCode)
                }
            }
        }()

        next.ServeHTTP(w, r)
    })
}

// 自定义错误类型
type MyBusinessError struct {
    StatusCode int
    Message    string
}

func (e *MyBusinessError) Error() string {
    return e.Message
}

// 从 context 获取请求 ID
func GetReqID(ctx context.Context) string {
    if reqID, ok := ctx.Value("requestID").(string); ok {
        return reqID
    }
    return ""
}
```

### 6.3 使用 panic 传递业务错误（不推荐但可行）

虽然不推荐使用 panic 处理预期内的业务错误，但有时为了简化多层调用的错误传播，可以这样做：

```go
func MustGetUser(userID string) *User {
    user, err := db.GetUser(userID)
    if err != nil {
        if err == sql.ErrNoRows {
            panic(&MyBusinessError{
                StatusCode: http.StatusNotFound,
                Message:    "User not found",
            })
        }
        panic(&MyBusinessError{
            StatusCode: http.StatusInternalServerError,
            Message:    "Database error",
        })
    }
    return user
}

// 使用示例（配合自定义 Recoverer）
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "id")
    
    // 如果用户不存在或数据库错误，会 panic
    // 自定义 Recoverer 会捕获并返回合适的响应
    user := MustGetUser(userID)
    
    json.NewEncoder(w).Encode(user)
}
```

**注意**：这种方式会让控制流变得不明显，建议谨慎使用。

## 7. 最佳实践建议

### 7.1 Recoverer 注册位置

**推荐**：尽可能早地注册 `Recoverer`，但要考虑日志中间件的顺序：

```go
r := chi.NewRouter()

// 1. 最先注册 RequestID（如果有）
r.Use(middleware.RequestID)

// 2. 然后注册 Logger（这样 panic 也能被记录）
r.Use(middleware.Logger)

// 3. 然后注册 Recoverer（捕获后续所有中间件和 handler 的 panic）
r.Use(middleware.Recoverer)

// 4. 其他中间件
r.Use(authMiddleware)
r.Use(corsMiddleware)

// 路由定义
r.Get("/", handler)
```

**为什么这样注册**：
- `RequestID`：为每个请求分配唯一 ID，方便追踪
- `Logger`：记录请求日志，即使发生 panic 也能记录
- `Recoverer`：捕获后续所有 panic

### 7.2 错误处理分层策略

推荐使用三层错误处理策略：

#### 第一层：预期内的业务错误 → http.Error()

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // 参数验证
    id := chi.URLParam(r, "id")
    if id == "" {
        http.Error(w, "id is required", http.StatusBadRequest)
        return
    }
    
    // 业务逻辑
    result, err := doSomething(id)
    if err != nil {
        if err == ErrNotFound {
            http.Error(w, "resource not found", http.StatusNotFound)
            return
        }
        if err == ErrPermissionDenied {
            http.Error(w, "permission denied", http.StatusForbidden)
            return
        }
        // 非预期错误，记录日志
        log.Printf("unexpected error: %v", err)
        http.Error(w, "internal server error", http.StatusInternalServerError)
        return
    }
    
    // 正常响应
    json.NewEncoder(w).Encode(result)
}
```

#### 第二层：非预期的运行时错误 → Panic + Recoverer

```go
// 这些错误应该被 Recoverer 捕获：
// - 空指针解引用
// - 数组越界
// - 类型断言失败
// - 并发 map 读写
// - 等等

// 示例：空指针解引用
func badHandler(w http.ResponseWriter, r *http.Request) {
    var data *Data
    fmt.Println(data.Name)  // panic: nil pointer dereference
}
```

#### 第三层：无法恢复的错误 → 让程序崩溃

```go
// 有些错误是无法恢复的，应该让程序崩溃：
// - 初始化失败（如数据库连接失败）
// - 配置错误
// - 资源耗尽（如内存不足）

// 示例：初始化失败
func main() {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        // 无法恢复，直接 panic 让程序崩溃
        log.Fatalf("failed to connect to database: %v", err)
    }
    // ...
}
```

### 7.3 中间件编写的最佳实践

编写中间件时，应该考虑 panic 的影响：

#### 推荐做法：

```go
func SafeMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 前置逻辑：使用 defer 确保资源清理
        var resource *Resource
        defer func() {
            if resource != nil {
                resource.Close()
            }
        }()
        
        // 可能发生 panic 的操作
        resource, err := acquireResource()
        if err != nil {
            http.Error(w, "failed to acquire resource", http.StatusServiceUnavailable)
            return
        }
        
        // 将资源注入 context
        ctx := context.WithValue(r.Context(), "resource", resource)
        next.ServeHTTP(w, r.WithContext(ctx))
        
        // 后置逻辑
        log.Println("request completed")
    })
}
```

#### 不推荐做法：

```go
func UnsafeMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        resource := acquireResource()  // 可能 panic
        // 如果上面 panic 了，resource 永远不会被释放
        
        next.ServeHTTP(w, r)
        
        resource.Close()  // 如果中间 panic，这里不会执行
    })
}
```

### 7.4 测试 Panic 恢复

编写测试确保 Recoverer 正常工作：

```go
func TestRecoverer(t *testing.T) {
    r := chi.NewRouter()
    r.Use(middleware.Recoverer)
    
    // 注册一个会 panic 的 handler
    r.Get("/panic", func(w http.ResponseWriter, r *http.Request) {
        panic("test panic")
    })
    
    // 创建测试服务器
    ts := httptest.NewServer(r)
    defer ts.Close()
    
    // 发送请求
    resp, err := http.Get(ts.URL + "/panic")
    if err != nil {
        t.Fatalf("failed to make request: %v", err)
    }
    defer resp.Body.Close()
    
    // 验证状态码
    if resp.StatusCode != http.StatusInternalServerError {
        t.Errorf("expected status 500, got %d", resp.StatusCode)
    }
    
    // 验证服务器没有崩溃（测试继续运行就说明没有崩溃）
    t.Log("Recoverer successfully caught the panic")
}

func TestRecovererDoesNotRecoverAbortHandler(t *testing.T) {
    defer func() {
        rcv := recover()
        if rcv != http.ErrAbortHandler {
            t.Fatalf("http.ErrAbortHandler should not be recovered, got: %v", rcv)
        }
    }()
    
    w := httptest.NewRecorder()
    r := chi.NewRouter()
    r.Use(middleware.Recoverer)
    
    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        panic(http.ErrAbortHandler)
    })
    
    req, _ := http.NewRequest("GET", "/", nil)
    r.ServeHTTP(w, req)
}
```

### 7.5 生产环境建议

1. **始终使用 Recoverer**：
   - 生产环境必须注册 `Recoverer` 中间件
   - 防止单个请求的 panic 导致整个服务崩溃

2. **自定义 Recoverer**：
   - 根据业务需求自定义响应格式
   - 集成结构化日志
   - 添加告警机制（如 panic 时发送通知）

3. **监控和告警**：
   - 监控 panic 发生的频率
   - 对 panic 设置告警阈值
   - 记录完整的堆栈信息便于排查

4. **开发环境 vs 生产环境**：
   - 开发环境：显示详细的错误信息和堆栈
   - 生产环境：隐藏内部细节，返回友好的错误信息

## 8. 常见问题解答

### Q1: Recoverer 能捕获 goroutine 中的 panic 吗？

**A: 不能**。`Recoverer` 只能捕获当前 goroutine 中的 panic。如果 handler 中启动了新的 goroutine 并发生 panic，这个 panic 不会被 `Recoverer` 捕获。

```go
func handler(w http.ResponseWriter, r *http.Request) {
    go func() {
        // 这个 panic 不会被 Recoverer 捕获！
        // 会导致整个程序崩溃
        panic("goroutine panic")
    }()
    
    w.Write([]byte("ok"))
}
```

**解决方案**：在 goroutine 内部使用 recover：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    go func() {
        defer func() {
            if rvr := recover(); rvr != nil {
                log.Printf("goroutine panic: %v", rvr)
                // 不要重新 panic，否则程序还是会崩溃
            }
        }()
        
        panic("goroutine panic")
    }()
    
    w.Write([]byte("ok"))
}
```

### Q2: 如果 ResponseWriter 已经被写入，Recoverer 还能改变状态码吗？

**A: 不能**。HTTP 协议规定状态码必须在响应体之前发送。如果 `WriteHeader` 已经被调用，或者 `Write` 已经被调用（会隐式调用 `WriteHeader(200)`），后续的 `WriteHeader` 调用会被忽略。

```go
func handler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(200)  // 状态码已发送
    w.Write([]byte("ok"))
    
    panic("something went wrong")  // Recoverer 尝试写 500，但会被忽略
}
```

### Q3: 如何区分预期错误和非预期错误？

**A: 参考以下原则**：

| 类型 | 处理方式 | 示例 |
|------|----------|------|
| **用户输入错误** | http.Error(400) | 参数缺失、格式错误 |
| **认证失败** | http.Error(401) | Token 无效、过期 |
| **权限不足** | http.Error(403) | 无权限访问资源 |
| **资源不存在** | http.Error(404) | 用户不存在、订单不存在 |
| **资源冲突** | http.Error(409) | 重复创建、乐观锁失败 |
| **服务不可用** | http.Error(503) | 下游服务超时、熔断 |
| **空指针解引用** | Panic + Recoverer | 程序 bug |
| **数组越界** | Panic + Recoverer | 程序 bug |
| **类型断言失败** | Panic + Recoverer | 程序 bug |
| **并发 map 读写** | Panic + Recoverer | 程序 bug |

### Q4: Recoverer 会影响性能吗？

**A: 影响很小**。`defer` 和 `recover` 在 Go 中是相对轻量级的操作。只有在发生 panic 时才会有额外的开销（堆栈收集、日志记录等），而 panic 在正常情况下应该很少发生。

**性能开销分析**：
- 正常路径：几乎无额外开销
- Panic 路径：收集堆栈、日志记录、恢复逻辑（这些都是必要的）

### Q5: 如何让 Recoverer 返回自定义的错误页面？

**A: 自定义 Recoverer 中间件**：

```go
func CustomRecoverer(errorPageHandler http.Handler) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if rvr := recover(); rvr != nil {
                    if rvr == http.ErrAbortHandler {
                        panic(rvr)
                    }
                    
                    // 记录日志
                    log.Printf("PANIC: %v\n%s", rvr, debug.Stack())
                    
                    // 使用自定义错误页面处理器
                    w.WriteHeader(http.StatusInternalServerError)
                    errorPageHandler.ServeHTTP(w, r)
                }
            }()
            
            next.ServeHTTP(w, r)
        })
    }
}

// 使用示例
errorPageHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    // 渲染 500 错误页面
    http.ServeFile(w, r, "./static/500.html")
})

r.Use(CustomRecoverer(errorPageHandler))
```

## 9. 总结

Chi 框架的 `middleware.Recoverer` 是一个强大的工具，能够保护 Web 服务免受意外 panic 的影响。理解其工作原理和限制对于构建健壮的服务至关重要。

### 核心要点回顾

1. **Recoverer 的工作原理**：
   - 使用 `defer recover()` 模式捕获 panic
   - 只能捕获其作用域内的 panic
   - 特殊处理 `http.ErrAbortHandler`

2. **捕获范围**：
   - ✅ 能捕获：Recoverer 之后的中间件、最终 handler
   - ❌ 不能捕获：Recoverer 之前的中间件、新启动的 goroutine

3. **与 http.Error() 的区别**：
   - Panic：非预期错误、中断流程、自动堆栈
   - http.Error()：预期错误、正常流程、完全可控

4. **最佳实践**：
   - 尽早注册 Recoverer（在 Logger 之后）
   - 分层错误处理策略
   - 自定义 Recoverer 满足业务需求
   - goroutine 内部自行处理 panic

### 架构建议

```
请求到达
    ↓
┌─────────────────────────────────────┐
│  RequestID (分配唯一请求 ID)         │
├─────────────────────────────────────┤
│  Logger (记录请求日志)               │
├─────────────────────────────────────┤
│  Recoverer (捕获 panic)             │  ← 安全边界
│  ┌───────────────────────────────┐  │
│  │  Auth (认证)                  │  │
│  ├───────────────────────────────┤  │
│  │  CORS (跨域处理)              │  │
│  ├───────────────────────────────┤  │
│  │  RateLimit (限流)             │  │
│  ├───────────────────────────────┤  │
│  │  Handler (业务逻辑)           │  │
│  │  - 预期错误: http.Error()    │  │
│  │  - 非预期错误: panic          │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
    ↓
响应返回
```

通过合理使用 `Recoverer` 和分层错误处理策略，可以构建出既健壮又易于维护的 Web 服务。

## 10. 参考源码位置

- `middleware/recoverer.go:22-49`：`Recoverer` 中间件核心实现
- `middleware/recoverer.go:54-64`：`PrintPrettyStack` 美化堆栈打印
- `chain.go:36-49`：`chain` 函数中间件组装逻辑
- `mux.go:63-92`：`ServeHTTP` 请求处理入口
