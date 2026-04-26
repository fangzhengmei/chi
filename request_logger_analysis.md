# Chi 框架请求日志中间件实现机制分析

## 1. 概述

Chi 框架提供了 `middleware.Logger` 和 `middleware.RequestLogger` 两个日志中间件，用于记录 HTTP 请求的详细信息。理解这些中间件如何与 `RouteContext` 交互、何时获取路由信息，以及如何自定义日志格式，对于构建可观测的 Web 服务至关重要。

本文档深入分析以下内容：
- `Logger` 和 `RequestLogger` 的关系与实现
- 日志中间件与 `RouteContext` 的交互机制
- 路由模式（route pattern）的写入时机
- `LogFormatter` 的自定义扩展点设计
- 如何在日志中获取路由信息

## 2. 日志中间件核心架构

### 2.1 组件关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                        日志中间件架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Logger()  ──────►  DefaultLogger (包级变量)                    │
│                          │                                      │
│                          ▼                                      │
│  RequestLogger(f) ────►  返回中间件函数                         │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              LogFormatter 接口                           │  │
│  │  ┌─────────────────────────────────────────────────┐    │  │
│  │  │ NewLogEntry(r *http.Request) LogEntry           │    │  │
│  │  └─────────────────────────────────────────────────┘    │  │
│  │                          │                               │  │
│  │          ┌───────────────┴───────────────┐              │  │
│  │          │                               │              │  │
│  │          ▼                               ▼              │  │
│  │  DefaultLogFormatter            自定义 LogFormatter    │  │
│  │  (默认实现)                        (用户可扩展)          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                LogEntry 接口                             │  │
│  │  ┌─────────────────────────────────────────────────┐    │  │
│  │  │ Write(status, bytes, header, elapsed, extra)   │    │  │
│  │  │ Panic(v interface{}, stack []byte)              │    │  │
│  │  └─────────────────────────────────────────────────┘    │  │
│  │                          │                               │  │
│  │          ┌───────────────┴───────────────┐              │  │
│  │          │                               │              │  │
│  │          ▼                               ▼              │  │
│  │  defaultLogEntry                自定义 LogEntry         │  │
│  │  (默认实现)                        (用户可扩展)          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心接口定义

```go
// LogFormatter 为每个请求初始化一个新的 LogEntry
type LogFormatter interface {
    NewLogEntry(r *http.Request) LogEntry
}

// LogEntry 在请求完成时记录最终日志
type LogEntry interface {
    Write(status, bytes int, header http.Header, elapsed time.Duration, extra interface{})
    Panic(v interface{}, stack []byte)
}
```
**middleware/logger.go:61-72**

### 2.3 Logger 与 RequestLogger 的关系

```go
// Logger 是一个简单的包装器，调用 DefaultLogger
func Logger(next http.Handler) http.Handler {
    return DefaultLogger(next)
}

// RequestLogger 使用自定义的 LogFormatter 返回一个日志处理器
func RequestLogger(f LogFormatter) func(next http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        fn := func(w http.ResponseWriter, r *http.Request) {
            entry := f.NewLogEntry(r)
            ww := NewWrapResponseWriter(w, r.ProtoMajor)

            t1 := time.Now()
            defer func() {
                entry.Write(ww.Status(), ww.BytesWritten(), ww.Header(), time.Since(t1), nil)
            }()

            next.ServeHTTP(ww, WithLogEntry(r, entry))
        }
        return http.HandlerFunc(fn)
    }
}
```
**middleware/logger.go:39-59**

### 2.4 包级变量初始化

```go
var (
    // LogEntryCtxKey 是存储请求日志条目的 context 键
    LogEntryCtxKey = &contextKey{"LogEntry"}

    // DefaultLogger 由 Logger 中间件处理器调用来记录每个请求
    // 它是包级变量，可以重新配置以实现自定义日志配置
    DefaultLogger func(next http.Handler) http.Handler
)

func init() {
    color := true
    if runtime.GOOS == "windows" {
        color = false
    }
    // 初始化 DefaultLogger 为使用 DefaultLogFormatter 的 RequestLogger
    DefaultLogger = RequestLogger(&DefaultLogFormatter{
        Logger: log.New(os.Stdout, "", log.LstdFlags), 
        NoColor: !color
    })
}
```
**middleware/logger.go:13-21, 166-172**

**关键理解**：
- `Logger()` 只是 `DefaultLogger` 的简单包装
- `DefaultLogger` 在 `init()` 中被初始化为 `RequestLogger(&DefaultLogFormatter{...})`
- 这意味着 `Logger()` 默认使用 `DefaultLogFormatter` 来格式化日志

## 3. 默认日志实现分析

### 3.1 DefaultLogFormatter

```go
// DefaultLogFormatter 是实现 LogFormatter 的简单日志器
type DefaultLogFormatter struct {
    Logger  LoggerInterface
    NoColor bool
}

// NewLogEntry 为请求创建一个新的 LogEntry
func (l *DefaultLogFormatter) NewLogEntry(r *http.Request) LogEntry {
    useColor := !l.NoColor
    entry := &defaultLogEntry{
        DefaultLogFormatter: l,
        request:             r,
        buf:                 &bytes.Buffer{},
        useColor:            useColor,
    }

    // 获取请求 ID（如果有）
    reqID := GetReqID(r.Context())
    if reqID != "" {
        cW(entry.buf, useColor, nYellow, "[%s] ", reqID)
    }
    
    // 构建请求信息（方法、URL、协议、远程地址）
    cW(entry.buf, useColor, nCyan, "\"")
    cW(entry.buf, useColor, bMagenta, "%s ", r.Method)

    scheme := "http"
    if r.TLS != nil {
        scheme = "https"
    }
    cW(entry.buf, useColor, nCyan, "%s://%s%s %s\" ", scheme, r.Host, r.RequestURI, r.Proto)

    entry.buf.WriteString("from ")
    entry.buf.WriteString(r.RemoteAddr)
    entry.buf.WriteString(" - ")

    return entry
}
```
**middleware/logger.go:91-125**

### 3.2 defaultLogEntry

```go
type defaultLogEntry struct {
    *DefaultLogFormatter
    request  *http.Request
    buf      *bytes.Buffer
    useColor bool
}

func (l *defaultLogEntry) Write(status, bytes int, header http.Header, elapsed time.Duration, extra interface{}) {
    // 根据状态码选择颜色
    switch {
    case status < 200:
        cW(l.buf, l.useColor, bBlue, "%03d", status)
    case status < 300:
        cW(l.buf, l.useColor, bGreen, "%03d", status)
    case status < 400:
        cW(l.buf, l.useColor, bCyan, "%03d", status)
    case status < 500:
        cW(l.buf, l.useColor, bYellow, "%03d", status)
    default:
        cW(l.buf, l.useColor, bRed, "%03d", status)
    }

    // 写入字节数
    cW(l.buf, l.useColor, bBlue, " %dB", bytes)

    // 写入耗时（根据耗时选择颜色）
    l.buf.WriteString(" in ")
    if elapsed < 500*time.Millisecond {
        cW(l.buf, l.useColor, nGreen, "%s", elapsed)
    } else if elapsed < 5*time.Second {
        cW(l.buf, l.useColor, nYellow, "%s", elapsed)
    } else {
        cW(l.buf, l.useColor, nRed, "%s", elapsed)
    }

    // 输出日志
    l.Logger.Print(l.buf.String())
}

func (l *defaultLogEntry) Panic(v interface{}, stack []byte) {
    PrintPrettyStack(v)
}
```
**middleware/logger.go:127-164**

### 3.3 默认日志的局限性

**重要发现**：默认的 `DefaultLogFormatter` 和 `defaultLogEntry` **并没有**从 `RouteContext` 中获取路由模式（route pattern）。

默认日志记录的信息包括：
- ✅ 请求 ID（如果有）
- ✅ HTTP 方法
- ✅ 完整的请求 URL（`r.RequestURI`）
- ✅ HTTP 协议版本
- ✅ 远程地址
- ✅ 状态码
- ✅ 响应字节数
- ✅ 请求耗时

**不包括**：
- ❌ 路由模式（如 `/users/{id}`）
- ❌ 路由参数值

这意味着默认的 `Logger` 中间件只能记录实际的请求路径（如 `/users/123`），而不能记录路由模式（如 `/users/{id}`）。

## 4. 日志中间件与 RouteContext 的关系

### 4.1 两个独立的 Context 系统

Chi 框架中有两个独立的 context 系统：

```
┌─────────────────────────────────────────────────────────────────┐
│                    请求 Context 结构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  r.Context() (http.Request 的 Context)                          │
│  │                                                              │
│  ├── RouteCtxKey ──────► *chi.Context (RouteContext)          │
│  │                         ├── RoutePatterns []string         │
│  │                         ├── URLParams RouteParams           │
│  │                         └── ...                             │
│  │                                                              │
│  └── LogEntryCtxKey ─────► middleware.LogEntry                │
│                            ├── Write(...)                      │
│                            └── Panic(...)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 关键工具函数

```go
// GetLogEntry 从请求中获取 context 中的 LogEntry
func GetLogEntry(r *http.Request) LogEntry {
    entry, _ := r.Context().Value(LogEntryCtxKey).(LogEntry)
    return entry
}

// WithLogEntry 设置请求 context 中的 LogEntry
func WithLogEntry(r *http.Request, entry LogEntry) *http.Request {
    r = r.WithContext(context.WithValue(r.Context(), LogEntryCtxKey, entry))
    return r
}
```
**middleware/logger.go:74-84**

```go
// RouteContext 从 http.Request Context 中返回 chi 的路由 Context 对象
func RouteContext(ctx context.Context) *Context {
    val, _ := ctx.Value(RouteCtxKey).(*Context)
    return val
}
```
**context.go:27-30**

### 4.3 为什么默认日志不包含路由模式

查看 `RequestLogger` 的实现：

```go
func RequestLogger(f LogFormatter) func(next http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        fn := func(w http.ResponseWriter, r *http.Request) {
            // 1. 请求开始时创建 LogEntry
            entry := f.NewLogEntry(r)  // ← 此时路由还未匹配！
            ww := NewWrapResponseWriter(w, r.ProtoMajor)

            t1 := time.Now()
            defer func() {
                // 3. 请求结束时写入日志
                entry.Write(ww.Status(), ww.BytesWritten(), ww.Header(), time.Since(t1), nil)
            }()

            // 2. 执行后续中间件和 handler（包括路由匹配）
            next.ServeHTTP(ww, WithLogEntry(r, entry))
        }
        return http.HandlerFunc(fn)
    }
}
```
**middleware/logger.go:44-59**

**时间线分析**：

```
请求到达
    │
    ▼
┌─────────────────┐
│ NewLogEntry(r)  │  ◄── 此时路由还未匹配！
│                 │      RouteContext 中没有路由信息
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ WrapResponse    │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ next.ServeHTTP  │  ◄── 路由匹配发生在这里！
│                 │      RoutePattern 被写入 RouteContext
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ defer 中的      │  ◄── 此时路由已匹配完成！
│ entry.Write()   │      可以从 RouteContext 获取路由信息
└─────────────────┘
    │
    ▼
请求结束
```

**关键问题**：
- `NewLogEntry(r)` 在请求开始时调用，此时路由还未匹配
- `entry.Write()` 在 `defer` 中调用，此时路由已匹配完成
- 但默认的 `defaultLogEntry.Write()` 并没有从 `RouteContext` 中获取路由信息

## 5. 路由模式的写入时机

### 5.1 RoutePatterns 的更新时机

路由模式是在路由匹配过程中写入 `RouteContext` 的。让我们回顾一下关键代码：

```go
func (n *node) FindRoute(rctx *Context, method methodTyp, path string) (*node, endpoints, http.Handler) {
    // 重置上下文的路由模式和参数
    rctx.routePattern = ""
    rctx.routeParams.Keys = rctx.routeParams.Keys[:0]
    rctx.routeParams.Values = rctx.routeParams.Values[:0]

    // 在路由树中查找匹配的处理器
    rn := n.findRoute(rctx, method, path)
    if rn == nil {
        return nil, nil, nil
    }

    // 将当前路由器的参数记录到请求生命周期的参数栈中
    rctx.URLParams.Keys = append(rctx.URLParams.Keys, rctx.routeParams.Keys...)
    rctx.URLParams.Values = append(rctx.URLParams.Values, rctx.routeParams.Values...)

    // 记录路由模式 ←── 关键点！
    if rn.endpoints[method].pattern != "" {
        rctx.routePattern = rn.endpoints[method].pattern
        rctx.RoutePatterns = append(rctx.RoutePatterns, rctx.routePattern)  // ←── 追加到切片
    }

    return rn, rn.endpoints, rn.endpoints[method].handler
}
```
**tree.go:374-397**

### 5.2 嵌套路由的模式累积

对于嵌套的子路由器，`RoutePatterns` 会累积所有层级的路由模式：

```go
// 示例路由结构
r := chi.NewRouter()
r.Route("/api", func(r chi.Router) {
    r.Route("/users", func(r chi.Router) {
        r.Get("/{id}", getUserHandler)
    })
})

// 请求 GET /api/users/123 时，RoutePatterns 会是：
// ["/api/*", "/users/*", "/{id}"]
```

### 5.3 RoutePattern() 方法

```go
// RoutePattern 构建特定请求的路由模式字符串
// 这个值在请求执行过程中会变化，建议在 next handler 之后使用
//
// 示例：
//	func Instrument(next http.Handler) http.Handler {
//		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
//			next.ServeHTTP(w, r)  // ←── 先执行
//			routePattern := chi.RouteContext(r.Context()).RoutePattern()  // ←── 再获取
//			measure(w, r, routePattern)
//		})
//	}
func (x *Context) RoutePattern() string {
    if x == nil {
        return ""
    }
    // 拼接所有路由模式
    routePattern := strings.Join(x.RoutePatterns, "")
    // 替换通配符
    routePattern = replaceWildcards(routePattern)
    // 清理末尾的斜杠
    if routePattern != "/" {
        routePattern = strings.TrimSuffix(routePattern, "//")
        routePattern = strings.TrimSuffix(routePattern, "/")
    }
    return routePattern
}
```
**context.go:109-134**

### 5.4 通配符替换

```go
// replaceWildcards 替换路由模式中的通配符
// 将 "/*/" 替换为 "/"，正确处理连续的通配符
func replaceWildcards(p string) string {
    for strings.Contains(p, "/*/") {
        p = strings.ReplaceAll(p, "/*/", "/")
    }
    return p
}
```
**context.go:139-144**

**示例**：
- 输入：`"/api/*/users/*/{id}"`
- 输出：`"/api/users/{id}"`

## 6. 如何在日志中获取路由信息

### 6.1 问题分析

默认的 `Logger` 不记录路由模式，但我们可以通过自定义 `LogFormatter` 和 `LogEntry` 来实现。

**关键时机**：
- `NewLogEntry()`：请求开始时调用，路由未匹配，无法获取路由模式
- `Write()`：请求结束时在 `defer` 中调用，路由已匹配，可以获取路由模式

### 6.2 自定义 LogFormatter 示例

```go
package main

import (
    "bytes"
    "log"
    "net/http"
    "os"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

// CustomLogFormatter 自定义日志格式化器
type CustomLogFormatter struct {
    Logger  middleware.LoggerInterface
    NoColor bool
}

// CustomLogEntry 自定义日志条目
type CustomLogEntry struct {
    *CustomLogFormatter
    request  *http.Request
    buf      *bytes.Buffer
    useColor bool
}

// NewLogEntry 实现 LogFormatter 接口
func (l *CustomLogFormatter) NewLogEntry(r *http.Request) middleware.LogEntry {
    useColor := !l.NoColor
    entry := &CustomLogEntry{
        CustomLogFormatter: l,
        request:             r,
        buf:                 &bytes.Buffer{},
        useColor:            useColor,
    }

    // 请求开始时记录基本信息（此时路由还未匹配）
    reqID := middleware.GetReqID(r.Context())
    if reqID != "" {
        entry.buf.WriteString("[")
        entry.buf.WriteString(reqID)
        entry.buf.WriteString("] ")
    }

    return entry
}

// Write 实现 LogEntry 接口
// 关键：此时路由已匹配，可以从 RouteContext 获取路由信息
func (l *CustomLogEntry) Write(status, bytes int, header http.Header, elapsed time.Duration, extra interface{}) {
    // 从 RouteContext 获取路由信息
    rctx := chi.RouteContext(l.request.Context())
    
    // 获取路由模式（如 /users/{id}）
    routePattern := ""
    if rctx != nil {
        routePattern = rctx.RoutePattern()
    }

    // 获取实际请求路径（如 /users/123）
    requestPath := l.request.URL.Path

    // 构建日志
    l.buf.WriteString("\"")
    l.buf.WriteString(l.request.Method)
    l.buf.WriteString(" ")
    
    // 记录路由模式（如果有）
    if routePattern != "" {
        l.buf.WriteString(routePattern)
    } else {
        l.buf.WriteString(requestPath)
    }
    
    l.buf.WriteString(" ")
    l.buf.WriteString(l.request.Proto)
    l.buf.WriteString("\" ")

    // 记录实际路径（用于对比）
    if routePattern != "" && routePattern != requestPath {
        l.buf.WriteString("(")
        l.buf.WriteString(requestPath)
        l.buf.WriteString(") ")
    }

    // 状态码
    switch {
    case status < 200:
        l.buf.WriteString("1xx")
    case status < 300:
        l.buf.WriteString("2xx")
    case status < 400:
        l.buf.WriteString("3xx")
    case status < 500:
        l.buf.WriteString("4xx")
    default:
        l.buf.WriteString("5xx")
    }
    l.buf.WriteString(" ")
    l.buf.WriteString(http.StatusText(status))
    l.buf.WriteString(" ")

    // 字节数
    l.buf.WriteString(string(rune(bytes)))
    l.buf.WriteString("B ")

    // 耗时
    l.buf.WriteString(elapsed.String())

    // 输出日志
    l.Logger.Print(l.buf.String())
}

// Panic 实现 LogEntry 接口
func (l *CustomLogEntry) Panic(v interface{}, stack []byte) {
    middleware.PrintPrettyStack(v)
}

// 使用示例
func main() {
    r := chi.NewRouter()

    // 使用自定义日志格式化器
    customLogger := middleware.RequestLogger(&CustomLogFormatter{
        Logger:  log.New(os.Stdout, "", log.LstdFlags),
        NoColor: false,
    })

    r.Use(customLogger)
    r.Use(middleware.RequestID)

    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello"))
    })

    r.Route("/api", func(r chi.Router) {
        r.Get("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
            userID := chi.URLParam(r, "id")
            w.Write([]byte("User: " + userID))
        })
    })

    http.ListenAndServe(":8080", r)
}
```

### 6.3 示例输出

请求 `GET /api/users/123` 时，日志输出可能如下：

```
2026/04/26 10:30:00 [req-123] "GET /api/users/{id} HTTP/1.1" (/api/users/123) 2xx OK 15B 1.234ms
```

可以看到：
- 路由模式：`/api/users/{id}`
- 实际路径：`/api/users/123`

## 7. LogFormatter 扩展点设计分析

### 7.1 设计模式

Chi 的日志中间件采用了**策略模式（Strategy Pattern）**：

- **Context**：`RequestLogger` 函数，负责编排日志流程
- **Strategy**：`LogFormatter` 接口，定义如何创建日志条目
- **Concrete Strategy**：`DefaultLogFormatter` 和用户自定义实现

### 7.2 扩展点分析

```
┌─────────────────────────────────────────────────────────────────┐
│                    扩展点设计架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  扩展点 1：包级变量 DefaultLogger                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ var DefaultLogger func(next http.Handler) http.Handler │  │
│  │                                                          │  │
│  │ 用途：全局替换默认的 Logger 中间件行为                   │  │
│  │ 示例：                                                    │  │
│  │   middleware.DefaultLogger = middleware.RequestLogger(  │  │
│  │       &MyCustomFormatter{},                              │  │
│  │   )                                                      │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  扩展点 2：LogFormatter 接口                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ type LogFormatter interface {                           │  │
│  │     NewLogEntry(r *http.Request) LogEntry              │  │
│  │ }                                                        │  │
│  │                                                          │  │
│  │ 用途：定义如何为每个请求创建日志条目                     │  │
│  │ 时机：请求开始时（路由匹配前）                           │  │
│  │ 可获取信息：r.Method, r.URL, r.Header, r.Context()     │  │
│  │ 不可获取：路由模式（还未匹配）                           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  扩展点 3：LogEntry 接口                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ type LogEntry interface {                               │  │
│  │     Write(status, bytes, header, elapsed, extra)        │  │
│  │     Panic(v, stack)                                      │  │
│  │ }                                                        │  │
│  │                                                          │  │
│  │ 用途：定义请求完成时如何写入日志                         │  │
│  │ 时机：请求结束时（defer 中，路由匹配后）                │  │
│  │ 可获取信息：                                              │  │
│  │   - 响应状态码、字节数、响应头、耗时                     │  │
│  │   - RouteContext 中的路由模式和参数                     │  │
│  │   - 请求的所有信息（通过保存的 request 引用）           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 高级自定义：结构化日志

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "os"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

// StructuredLogEntry 结构化日志条目
type StructuredLogEntry struct {
    request *http.Request
    logger  *log.Logger
}

// StructuredLogFormatter 结构化日志格式化器
type StructuredLogFormatter struct {
    Logger *log.Logger
}

func (f *StructuredLogFormatter) NewLogEntry(r *http.Request) middleware.LogEntry {
    return &StructuredLogEntry{
        request: r,
        logger:  f.Logger,
    }
}

// LogFields 日志字段结构
type LogFields struct {
    Timestamp    string            `json:"@timestamp"`
    RequestID    string            `json:"request_id,omitempty"`
    Method       string            `json:"method"`
    Path         string            `json:"path"`
    RoutePattern string            `json:"route_pattern,omitempty"`
    Protocol     string            `json:"protocol"`
    StatusCode   int               `json:"status_code"`
    StatusText   string            `json:"status_text"`
    BytesWritten int               `json:"bytes_written"`
    Duration     int64             `json:"duration_ms"`
    RemoteAddr   string            `json:"remote_addr"`
    UserAgent    string            `json:"user_agent,omitempty"`
    Referer      string            `json:"referer,omitempty"`
    URLParams    map[string]string `json:"url_params,omitempty"`
}

func (e *StructuredLogEntry) Write(status, bytes int, header http.Header, elapsed time.Duration, extra interface{}) {
    rctx := chi.RouteContext(e.request.Context())
    
    fields := LogFields{
        Timestamp:    time.Now().Format(time.RFC3339Nano),
        RequestID:    middleware.GetReqID(e.request.Context()),
        Method:       e.request.Method,
        Path:         e.request.URL.Path,
        Protocol:     e.request.Proto,
        StatusCode:   status,
        StatusText:   http.StatusText(status),
        BytesWritten: bytes,
        Duration:     elapsed.Milliseconds(),
        RemoteAddr:   e.request.RemoteAddr,
        UserAgent:    e.request.UserAgent(),
        Referer:      e.request.Referer(),
    }

    // 获取路由模式
    if rctx != nil {
        fields.RoutePattern = rctx.RoutePattern()
        
        // 获取 URL 参数
        if len(rctx.URLParams.Keys) > 0 {
            fields.URLParams = make(map[string]string)
            for i, key := range rctx.URLParams.Keys {
                if i < len(rctx.URLParams.Values) {
                    fields.URLParams[key] = rctx.URLParams.Values[i]
                }
            }
        }
    }

    // 序列化为 JSON
    jsonBytes, err := json.Marshal(fields)
    if err != nil {
        e.logger.Printf("Failed to marshal log fields: %v", err)
        return
    }

    e.logger.Println(string(jsonBytes))
}

func (e *StructuredLogEntry) Panic(v interface{}, stack []byte) {
    // 记录 panic 日志
    fields := LogFields{
        Timestamp:  time.Now().Format(time.RFC3339Nano),
        RequestID:  middleware.GetReqID(e.request.Context()),
        Method:     e.request.Method,
        Path:       e.request.URL.Path,
        StatusCode: http.StatusInternalServerError,
        StatusText: "PANIC",
    }
    
    jsonBytes, _ := json.Marshal(fields)
    e.logger.Println(string(jsonBytes))
    
    // 打印堆栈
    middleware.PrintPrettyStack(v)
}

// 使用示例
func main() {
    logger := log.New(os.Stdout, "", 0)
    
    r := chi.NewRouter()
    
    // 使用结构化日志
    r.Use(middleware.RequestLogger(&StructuredLogFormatter{
        Logger: logger,
    }))
    r.Use(middleware.RequestID)
    
    r.Get("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"status": "ok"}`))
    })
    
    http.ListenAndServe(":8080", r)
}
```

### 7.4 结构化日志输出示例

请求 `GET /users/123` 时的 JSON 日志输出：

```json
{
    "@timestamp": "2026-04-26T10:30:00.123456789Z",
    "request_id": "hostname-abc123-000001",
    "method": "GET",
    "path": "/users/123",
    "route_pattern": "/users/{id}",
    "protocol": "HTTP/1.1",
    "status_code": 200,
    "status_text": "OK",
    "bytes_written": 17,
    "duration_ms": 2,
    "remote_addr": "192.168.1.100:54321",
    "user_agent": "curl/7.68.0",
    "url_params": {
        "id": "123"
    }
}
```

## 8. 中间件注册顺序的影响

### 8.1 推荐的中间件顺序

```go
r := chi.NewRouter()

// 1. 最先：RequestID（为请求分配唯一标识）
r.Use(middleware.RequestID)

// 2. 其次：Logger（记录整个请求生命周期）
// 注意：Logger 应该在 Recoverer 之前，这样即使发生 panic 也能被记录
r.Use(middleware.Logger)

// 3. 然后：Recoverer（捕获 panic）
r.Use(middleware.Recoverer)

// 4. 其他业务中间件
r.Use(authMiddleware)
r.Use(corsMiddleware)

// 路由定义
r.Get("/", handler)
```

### 8.2 为什么 Logger 应该在 Recoverer 之前

查看 `RequestLogger` 和 `Recoverer` 的实现：

```go
// RequestLogger 使用 defer 在请求结束时写入日志
func RequestLogger(f LogFormatter) func(next http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        fn := func(w http.ResponseWriter, r *http.Request) {
            entry := f.NewLogEntry(r)
            ww := NewWrapResponseWriter(w, r.ProtoMajor)
            
            t1 := time.Now()
            defer func() {
                // 即使发生 panic，defer 也会执行
                entry.Write(ww.Status(), ww.BytesWritten(), ww.Header(), time.Since(t1), nil)
            }()
            
            next.ServeHTTP(ww, WithLogEntry(r, entry))
        }
        return http.HandlerFunc(fn)
    }
}
```

```go
// Recoverer 也使用 defer 捕获 panic
func Recoverer(next http.Handler) http.Handler {
    fn := func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rvr := recover(); rvr != nil {
                // 捕获 panic 并记录
                logEntry := GetLogEntry(r)
                if logEntry != nil {
                    logEntry.Panic(rvr, debug.Stack())
                } else {
                    PrintPrettyStack(rvr)
                }
                
                // 写入 500 状态码
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

### 8.3 顺序分析

#### 情况 1：Logger 在 Recoverer 之前（推荐）

```go
r.Use(Logger)      // 先注册
r.Use(Recoverer)   // 后注册
```

**执行顺序**（洋葱模型）：

```
请求到达
    │
    ▼
Logger 前置逻辑
    ├── 创建 LogEntry
    ├── 包装 ResponseWriter
    └── defer entry.Write()
    │
    ▼
Recoverer 前置逻辑
    └── defer recover()
    │
    ▼
Handler 执行 → 可能发生 panic
    │
    ▼（如果发生 panic）
Recoverer 的 defer recover() 执行
    ├── 捕获 panic
    ├── 调用 logEntry.Panic()
    └── 写入 500 状态码
    │
    ▼
Logger 的 defer entry.Write() 执行
    └── 记录日志（状态码 500）
    │
    ▼
响应返回
```

**优点**：
- ✅ Recoverer 可以通过 `GetLogEntry(r)` 获取 Logger 创建的 `LogEntry`
- ✅ `logEntry.Panic()` 会被调用，记录 panic 信息
- ✅ 最终的 `entry.Write()` 会记录正确的 500 状态码

#### 情况 2：Logger 在 Recoverer 之后（不推荐）

```go
r.Use(Recoverer)   // 先注册
r.Use(Logger)      // 后注册
```

**执行顺序**：

```
请求到达
    │
    ▼
Recoverer 前置逻辑
    └── defer recover()
    │
    ▼
Logger 前置逻辑
    ├── 创建 LogEntry
    ├── 包装 ResponseWriter
    └── defer entry.Write()
    │
    ▼
Handler 执行 → 发生 panic
    │
    ▼
Logger 的 defer entry.Write() 执行
    └── 记录日志（状态码可能是 0 或 200）
    │
    ▼
Recoverer 的 defer recover() 执行
    ├── 捕获 panic
    ├── GetLogEntry(r) 返回 nil（因为 Logger 已返回）
    ├── 直接调用 PrintPrettyStack()
    └── 写入 500 状态码（但响应可能已发送）
    │
    ▼
响应返回
```

**缺点**：
- ❌ Recoverer 无法获取 `LogEntry`
- ❌ 日志中记录的状态码可能不正确
- ❌ `logEntry.Panic()` 不会被调用

### 8.4 关键结论

**Logger 必须在 Recoverer 之前注册**，原因：

1. **`LogEntry` 的可见性**：Recoverer 需要通过 `GetLogEntry(r)` 获取 Logger 创建的 `LogEntry`
2. **`Panic()` 方法调用**：Recoverer 会调用 `logEntry.Panic()` 来记录 panic 信息
3. **状态码正确性**：Recoverer 写入 500 状态码后，Logger 的 `defer entry.Write()` 才能记录正确的状态码

## 9. 完整的请求日志流程图

### 9.1 正常请求流程

```
请求到达
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. RequestID 中间件                                             │
│    ├── 从 Header 获取或生成 RequestID                           │
│    └── 注入到 request.Context                                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Logger (RequestLogger) 中间件                                │
│    ├── entry = formatter.NewLogEntry(r)                        │
│    │       └── 记录请求基本信息（方法、URL、协议、远程地址）   │
│    ├── ww = NewWrapResponseWriter(w, proto)                    │
│    │       └── 包装 ResponseWriter 以捕获状态码和字节数        │
│    ├── defer func() {                                           │
│    │       entry.Write(ww.Status(), ww.BytesWritten(), ...)   │
│    │   }()                                                       │
│    └── r = WithLogEntry(r, entry)                               │
│            └── 将 LogEntry 注入到 request.Context              │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Recoverer 中间件                                             │
│    └── defer func() {                                           │
│           if rvr := recover(); rvr != nil {                    │
│               logEntry := GetLogEntry(r)  // 获取 LogEntry    │
│               if logEntry != nil {                              │
│                   logEntry.Panic(rvr, debug.Stack())           │
│               } else {                                          │
│                   PrintPrettyStack(rvr)                         │
│               }                                                 │
│               w.WriteHeader(500)                                │
│           }                                                     │
│       }()                                                       │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 其他中间件（Auth, CORS 等）                                  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 路由匹配 & Handler 执行                                       │
│    ├── tree.FindRoute(rctx, method, path)                      │
│    │       ├── 匹配路由                                          │
│    │       ├── 提取 URL 参数到 rctx.URLParams                  │
│    │       └── 追加路由模式到 rctx.RoutePatterns               │
│    ├── 注入参数到 http.Request (r.SetPathValue)                │
│    └── handler.ServeHTTP(w, r)                                  │
│            └── 执行业务逻辑                                      │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 中间件后置逻辑（按相反顺序）                                  │
│    │                                                             │
│    ├── 其他中间件后置逻辑                                        │
│    │                                                             │
│    ├── Recoverer 后置逻辑                                       │
│    │       └── (没有 panic，defer 什么也不做)                  │
│    │                                                             │
│    └── Logger 后置逻辑（defer entry.Write()）                  │
│            ├── 此时可以从 RouteContext 获取：                   │
│            │   ├── rctx.RoutePattern() → 路由模式             │
│            │   └── rctx.URLParams → URL 参数                   │
│            ├── 记录：状态码、字节数、耗时                       │
│            └── 输出日志                                          │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
响应返回
```

### 9.2 发生 Panic 的请求流程

```
请求到达
    │
    ▼
1. RequestID → 2. Logger → 3. Recoverer → 4. 其他中间件 → 5. Handler
    │
    ▼（Handler 中发生 panic）
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Panic 向上传播                                                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 其他中间件：跳过，panic 继续传播                                 │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Recoverer 的 defer recover() 执行                              │
│    ├── 捕获 panic: rvr = recover()                             │
│    ├── logEntry = GetLogEntry(r)  ←── 获取 Logger 创建的     │
│    │                                          LogEntry          │
│    ├── logEntry.Panic(rvr, debug.Stack())  ←── 记录 panic    │
│    └── w.WriteHeader(500)  ←── 设置 500 状态码               │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Logger 的 defer entry.Write() 执行                             │
│    ├── ww.Status() → 500 (由 Recoverer 设置)                  │
│    ├── 获取路由信息（如果路由已匹配）                           │
│    └── 输出日志（包含 500 状态码）                              │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
响应返回（状态码 500）
```

## 10. 最佳实践建议

### 10.1 中间件注册顺序

```go
// 推荐顺序
r := chi.NewRouter()

// 1. RequestID：最先分配唯一标识
r.Use(middleware.RequestID)

// 2. Logger：其次记录请求（必须在 Recoverer 之前）
r.Use(middleware.Logger)

// 3. Recoverer：然后捕获 panic
r.Use(middleware.Recoverer)

// 4. 其他业务中间件
r.Use(authMiddleware)
r.Use(corsMiddleware)
r.Use(rateLimitMiddleware)
```

### 10.2 自定义日志的最佳实践

1. **在 `Write()` 中获取路由信息**：
   ```go
   func (e *MyLogEntry) Write(status, bytes int, header http.Header, elapsed time.Duration, extra interface{}) {
       // 此时路由已匹配，可以安全获取
       rctx := chi.RouteContext(e.request.Context())
       if rctx != nil {
           routePattern := rctx.RoutePattern()
           // 使用路由模式...
       }
   }
   ```

2. **处理 404 路由**：
   - 404 请求不会匹配任何路由，`RoutePatterns` 为空
   - 在日志中应该记录实际的请求路径

3. **结构化日志**：
   - 生产环境推荐使用结构化日志（JSON 格式）
   - 包含路由模式、URL 参数、响应头字段等元数据

### 10.3 日志字段建议

| 字段 | 来源 | 说明 |
|------|------|------|
| `@timestamp` | `time.Now()` | 日志时间戳 |
| `request_id` | `middleware.GetReqID()` | 请求唯一标识 |
| `method` | `r.Method` | HTTP 方法 |
| `path` | `r.URL.Path` | 实际请求路径 |
| `route_pattern` | `rctx.RoutePattern()` | 路由模式（如 `/users/{id}`） |
| `protocol` | `r.Proto` | HTTP 协议版本 |
| `status_code` | `ww.Status()` | HTTP 状态码 |
| `bytes_written` | `ww.BytesWritten()` | 响应字节数 |
| `duration_ms` | `elapsed.Milliseconds()` | 请求耗时（毫秒） |
| `remote_addr` | `r.RemoteAddr` | 客户端地址 |
| `user_agent` | `r.UserAgent()` | 用户代理 |
| `referer` | `r.Referer()` | 来源页面 |
| `url_params` | `rctx.URLParams` | URL 参数键值对 |

### 10.4 监控与告警

基于日志字段可以设置以下监控：

1. **错误率监控**：
   - 监控 `status_code` 为 5xx 的请求比例
   - 设置告警阈值（如 > 1%）

2. **响应时间监控**：
   - 监控 `duration_ms` 的 p95、p99 值
   - 设置慢请求告警

3. **路由级别监控**：
   - 按 `route_pattern` 分组统计
   - 发现特定路由的性能问题

4. **Panic 监控**：
   - 监控 `logEntry.Panic()` 的调用次数
   - Panic 通常表示程序 bug，需要立即关注

## 11. 常见问题解答

### Q1: 为什么默认的 Logger 不记录路由模式？

**A**: 这是设计决策。默认的 `DefaultLogFormatter` 被设计为简单、轻量级的实现。记录路由模式需要：

1. 在 `NewLogEntry()` 中保存 `*http.Request` 的引用
2. 在 `Write()` 中从 `RouteContext` 获取路由信息

这些步骤会增加一些复杂性，所以默认实现没有包含。但通过自定义 `LogFormatter` 可以轻松添加。

### Q2: 如何在中间件中获取路由模式？

**A**: 必须在 `next.ServeHTTP()` 之后获取，因为路由匹配发生在 `next.ServeHTTP()` 内部。

```go
func MyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // ❌ 这里路由还未匹配，获取不到
        // rctx := chi.RouteContext(r.Context())
        // pattern := rctx.RoutePattern()  // 空字符串
        
        start := time.Now()
        
        // 先执行后续中间件和 handler
        next.ServeHTTP(w, r)
        
        // ✅ 这里路由已匹配，可以获取
        rctx := chi.RouteContext(r.Context())
        pattern := rctx.RoutePattern()
        
        log.Printf("Route: %s, Duration: %v", pattern, time.Since(start))
    })
}
```

### Q3: 404 请求的路由模式是什么？

**A**: 404 请求不会匹配任何路由，所以 `RoutePatterns` 为空，`RoutePattern()` 返回空字符串。

```go
// 对于 404 请求
rctx := chi.RouteContext(r.Context())
rctx.RoutePatterns  // []string{} (空切片)
rctx.RoutePattern() // "" (空字符串)
```

**处理建议**：
```go
func (e *MyLogEntry) Write(status, bytes int, header http.Header, elapsed time.Duration, extra interface{}) {
    rctx := chi.RouteContext(e.request.Context())
    
    routePattern := ""
    if rctx != nil {
        routePattern = rctx.RoutePattern()
    }
    
    // 如果没有路由模式，使用实际路径
    displayPath := routePattern
    if displayPath == "" {
        displayPath = e.request.URL.Path
    }
    
    // ... 记录日志
}
```

### Q4: 子路由器中的路由模式会被正确拼接吗？

**A**: 会的。`RoutePatterns` 会累积所有层级的路由模式，`RoutePattern()` 方法会正确拼接并处理通配符。

```go
// 路由结构
r.Route("/api", func(r chi.Router) {
    r.Route("/v1", func(r chi.Router) {
        r.Get("/users/{id}", handler)
    })
})

// 请求 GET /api/v1/users/123
rctx.RoutePatterns  // ["/api/*", "/v1/*", "/users/{id}"]
rctx.RoutePattern() // "/api/v1/users/{id}" (通配符已被替换)
```

### Q5: 如何在 Panic 时获取路由信息？

**A**: 在 `LogEntry.Panic()` 方法中同样可以获取路由信息，但需要注意：

1. 如果 panic 发生在路由匹配**之前**，`RoutePatterns` 为空
2. 如果 panic 发生在路由匹配**之后**，`RoutePatterns` 有值

```go
func (e *MyLogEntry) Panic(v interface{}, stack []byte) {
    rctx := chi.RouteContext(e.request.Context())
    
    routePattern := ""
    if rctx != nil {
        routePattern = rctx.RoutePattern()
    }
    
    // 记录 panic 日志，包含路由模式（如果有）
    log.Printf("PANIC: %v\nRoute: %s\nStack:\n%s", v, routePattern, stack)
}
```

## 12. 总结

Chi 框架的请求日志中间件设计精巧且高度可扩展。理解其工作原理对于构建可观测的 Web 服务至关重要。

### 核心要点回顾

1. **Logger 与 RequestLogger 的关系**：
   - `Logger()` 是 `DefaultLogger` 的简单包装
   - `DefaultLogger` 在 `init()` 中被初始化为 `RequestLogger(&DefaultLogFormatter{...})`
   - `RequestLogger()` 是工厂函数，接收 `LogFormatter` 并返回中间件

2. **默认日志的局限性**：
   - 默认的 `DefaultLogFormatter` **不记录**路由模式
   - 只记录实际请求路径、状态码、字节数、耗时等基本信息

3. **路由模式的写入时机**：
   - 路由模式在 `tree.FindRoute()` 中被追加到 `RouteContext.RoutePatterns`
   - `RoutePattern()` 方法在请求结束时拼接所有模式并处理通配符
   - **必须**在 `next.ServeHTTP()` 之后获取

4. **自定义扩展点**：
   - **包级变量** `DefaultLogger`：全局替换默认行为
   - **接口** `LogFormatter`：定义如何创建日志条目
   - **接口** `LogEntry`：定义如何写入日志（`Write()` 和 `Panic()`）

5. **中间件顺序至关重要**：
   - `Logger` 必须在 `Recoverer` **之前**注册
   - 这样 `Recoverer` 才能获取 `LogEntry` 并调用 `Panic()`
   - 这样才能记录正确的 500 状态码

### 架构优势

Chi 的日志中间件设计体现了以下优势：

1. **关注点分离**：
   - `RequestLogger` 负责编排流程
   - `LogFormatter` 负责创建日志条目
   - `LogEntry` 负责写入日志

2. **高度可扩展**：
   - 通过实现接口可以完全自定义日志格式
   - 支持结构化日志、JSON 输出等

3. **与 RouteContext 解耦**：
   - 默认实现不依赖路由信息，保持简单
   - 自定义实现可以按需获取路由信息

4. **Panic 感知**：
   - `LogEntry.Panic()` 方法允许在 panic 时记录额外信息
   - 与 `Recoverer` 中间件无缝配合

### 最终建议

- **开发环境**：可以使用默认的 `Logger`，彩色输出便于调试
- **生产环境**：自定义 `LogFormatter` 实现结构化日志，包含路由模式、URL 参数等丰富字段
- **中间件顺序**：严格遵循 `RequestID → Logger → Recoverer → 业务中间件` 的顺序
- **监控告警**：基于日志字段设置错误率、响应时间、Panic 次数等监控指标

## 13. 参考源码位置

- `middleware/logger.go:39-41`：`Logger` 函数实现
- `middleware/logger.go:44-59`：`RequestLogger` 函数实现
- `middleware/logger.go:61-72`：`LogFormatter` 和 `LogEntry` 接口定义
- `middleware/logger.go:91-125`：`DefaultLogFormatter` 实现
- `middleware/logger.go:134-164`：`defaultLogEntry` 实现
- `middleware/logger.go:166-172`：`DefaultLogger` 初始化
- `middleware/recoverer.go:22-49`：`Recoverer` 中间件实现
- `context.go:109-134`：`RoutePattern()` 方法实现
- `tree.go:374-397`：`FindRoute()` 中路由模式的写入
