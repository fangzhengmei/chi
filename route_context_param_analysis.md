# Chi 框架路由上下文与参数传递机制分析

## 1. 概述

Chi 框架通过 `RouteContext` 实现了灵活且高效的路由参数传递机制。本文档深入分析 URL 参数（如 `/users/{id}` 中的 `id`）和通配符匹配结果的流转过程，包括 `RouteContext` 的创建时机、路由匹配后参数的注入方式，以及 `chi.URLParam()` 的底层实现原理。

## 2. 核心数据结构

### 2.1 Context 结构体

`Context` 是 Chi 框架的路由上下文核心数据结构：

```go
type Context struct {
    Routes Routes

    // parentCtx is the parent of this one, for using Context as a
    // context.Context directly. This is an optimization that saves
    // 1 allocation.
    parentCtx context.Context

    // Routing path/method override used during the route search.
    // See Mux#routeHTTP method.
    RoutePath   string
    RouteMethod string

    // URLParams are the stack of routeParams captured during the
    // routing lifecycle across a stack of sub-routers.
    URLParams RouteParams

    // Route parameters matched for the current sub-router. It is
    // intentionally unexported so it can't be tampered.
    routeParams RouteParams

    // The endpoint routing pattern that matched the request URI path
    // or `RoutePath` of the current sub-router. This value will update
    // during the lifecycle of a request passing through a stack of
    // sub-routers.
    routePattern string

    // Routing pattern stack throughout the lifecycle of the request,
    // across all connected routers. It is a record of all matching
    // patterns across a stack of sub-routers.
    RoutePatterns []string

    methodsAllowed   []methodTyp // allowed methods in case of a 405
    methodNotAllowed bool
}
```
**context.go:45-79**

### 2.2 RouteParams 结构体

用于高效存储 URL 路由参数的键值对：

```go
type RouteParams struct {
    Keys, Values []string
}

// Add will append a URL parameter to the end of the route param
func (s *RouteParams) Add(key, value string) {
    s.Keys = append(s.Keys, key)
    s.Values = append(s.Values, value)
}
```
**context.go:146-155**

### 2.3 关键设计要点

1. **双参数存储**：
   - `URLParams`：跨子路由器的参数栈，包含所有层级路由器捕获的参数
   - `routeParams`：当前子路由器匹配的参数，未导出防止被篡改

2. **路径模式追踪**：
   - `routePattern`：当前子路由器匹配的路径模式
   - `RoutePatterns`：所有子路由器匹配模式的记录栈

3. **上下文复用**：
   - 使用 `sync.Pool` 进行对象池管理，减少内存分配
   - 通过 `Reset()` 方法重置状态以便复用

## 3. RouteContext 的创建时机

### 3.1 创建流程

`RouteContext` 的创建发生在请求处理的入口点 `ServeHTTP` 方法中：

```go
func (mx *Mux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 确保 mux 上定义了一些路由
    if mx.handler == nil {
        mx.NotFoundHandler().ServeHTTP(w, r)
        return
    }

    // 检查是否已存在来自父路由器的路由上下文
    rctx, _ := r.Context().Value(RouteCtxKey).(*Context)
    if rctx != nil {
        mx.handler.ServeHTTP(w, r)
        return
    }

    // 从 sync 池获取 RouteContext 对象
    rctx = mx.pool.Get().(*Context)
    rctx.Reset()
    rctx.Routes = mx
    rctx.parentCtx = r.Context()

    // 将 rctx 注入到请求的 context 中
    r = r.WithContext(context.WithValue(r.Context(), RouteCtxKey, rctx))

    // 处理请求，完成后将路由上下文放回池中
    mx.handler.ServeHTTP(w, r)
    mx.pool.Put(rctx)
}
```
**mux.go:63-92**

### 3.2 关键步骤分析

1. **上下文检查**：
   - 首先检查请求的 context 中是否已存在 `RouteContext`
   - 如果存在（来自父路由器），则直接使用，避免重复创建

2. **对象池获取**：
   - 从 `sync.Pool` 中获取一个 `Context` 对象
   - 这是一种性能优化，避免频繁创建和销毁对象

3. **状态重置**：
   - 调用 `Reset()` 方法重置所有状态字段
   - 确保对象是"干净"的，可以安全复用

4. **上下文注入**：
   - 将 `RouteContext` 注入到请求的 `context` 中
   - 使用 `RouteCtxKey` 作为键，这是一个指向 `contextKey` 的指针

5. **请求处理完成**：
   - 处理完成后，将 `RouteContext` 放回对象池

### 3.3 Reset 方法实现

```go
func (x *Context) Reset() {
    x.Routes = nil
    x.RoutePath = ""
    x.RouteMethod = ""
    x.RoutePatterns = x.RoutePatterns[:0]
    x.URLParams.Keys = x.URLParams.Keys[:0]
    x.URLParams.Values = x.URLParams.Values[:0]

    x.routePattern = ""
    x.routeParams.Keys = x.routeParams.Keys[:0]
    x.routeParams.Values = x.routeParams.Values[:0]
    x.methodNotAllowed = false
    x.methodsAllowed = x.methodsAllowed[:0]
    x.parentCtx = nil
}
```
**context.go:82-96**

**设计要点**：
- 切片使用 `[:0]` 方式重置，保留底层数组，避免重新分配
- 所有字段都被显式重置，确保对象状态干净

## 4. 路由匹配与参数提取

### 4.1 路由匹配入口

路由匹配发生在 `routeHTTP` 方法中：

```go
func (mx *Mux) routeHTTP(w http.ResponseWriter, r *http.Request) {
    // 从请求 context 中获取路由上下文
    rctx := r.Context().Value(RouteCtxKey).(*Context)

    // 获取请求路由路径
    routePath := rctx.RoutePath
    if routePath == "" {
        if r.URL.RawPath != "" {
            routePath = r.URL.RawPath
        } else {
            routePath = r.URL.Path
        }
        if routePath == "" {
            routePath = "/"
        }
    }

    // 检查 HTTP 方法
    if rctx.RouteMethod == "" {
        rctx.RouteMethod = r.Method
    }
    method, ok := methodMap[rctx.RouteMethod]
    if !ok {
        mx.MethodNotAllowedHandler().ServeHTTP(w, r)
        return
    }

    // 查找路由
    if _, _, h := mx.tree.FindRoute(rctx, method, routePath); h != nil {
        // 将路由上下文的参数注入到 http.Request
        for i, key := range rctx.URLParams.Keys {
            value := rctx.URLParams.Values[i]
            r.SetPathValue(key, value)
        }
        r.Pattern = rctx.routePattern

        h.ServeHTTP(w, r)
        return
    }
    
    // 处理 404 或 405
    if rctx.methodNotAllowed {
        mx.MethodNotAllowedHandler(rctx.methodsAllowed...).ServeHTTP(w, r)
    } else {
        mx.NotFoundHandler().ServeHTTP(w, r)
    }
}
```
**mux.go:441-485**

### 4.2 FindRoute 方法分析

`FindRoute` 是路由匹配的核心方法，负责在路由树中查找匹配的路由并提取参数：

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

    // 记录路由模式
    if rn.endpoints[method].pattern != "" {
        rctx.routePattern = rn.endpoints[method].pattern
        rctx.RoutePatterns = append(rctx.RoutePatterns, rctx.routePattern)
    }

    return rn, rn.endpoints, rn.endpoints[method].handler
}
```
**tree.go:374-397**

### 4.3 递归路由匹配 findRoute

`findRoute` 是实际执行路由匹配的递归方法：

```go
func (n *node) findRoute(rctx *Context, method methodTyp, path string) *node {
    nn := n
    search := path

    for t, nds := range nn.children {
        ntyp := nodeTyp(t)
        if len(nds) == 0 {
            continue
        }

        var xn *node
        xsearch := search

        var label byte
        if search != "" {
            label = search[0]
        }

        switch ntyp {
        case ntStatic:
            // 静态节点匹配
            xn = nds.findEdge(label)
            if xn == nil || !strings.HasPrefix(xsearch, xn.prefix) {
                continue
            }
            xsearch = xsearch[len(xn.prefix):]

        case ntParam, ntRegexp:
            // 参数节点或正则表达式节点匹配
            if xsearch == "" {
                continue
            }

            for _, xn = range nds {
                // 查找参数分隔符
                p := strings.IndexByte(xsearch, xn.tail)

                if p < 0 {
                    if xn.tail == '/' {
                        p = len(xsearch)
                    } else {
                        continue
                    }
                } else if ntyp == ntRegexp && p == 0 {
                    continue
                }

                // 正则表达式验证
                if ntyp == ntRegexp && xn.rex != nil {
                    if !xn.rex.MatchString(xsearch[:p]) {
                        continue
                    }
                } else if strings.IndexByte(xsearch[:p], '/') != -1 {
                    // 避免跨路径段匹配
                    continue
                }

                // 记录参数值
                prevlen := len(rctx.routeParams.Values)
                rctx.routeParams.Values = append(rctx.routeParams.Values, xsearch[:p])
                xsearch = xsearch[p:]

                // 检查是否到达叶子节点
                if len(xsearch) == 0 {
                    if xn.isLeaf() {
                        h := xn.endpoints[method]
                        if h != nil && h.handler != nil {
                            // 记录参数键
                            rctx.routeParams.Keys = append(rctx.routeParams.Keys, h.paramKeys...)
                            return xn
                        }
                        
                        // 处理 405 情况
                        for endpoints := range xn.endpoints {
                            if endpoints == mALL || endpoints == mSTUB {
                                continue
                            }
                            rctx.methodsAllowed = append(rctx.methodsAllowed, endpoints)
                        }
                        rctx.methodNotAllowed = true
                    }
                }

                // 递归查找下一个节点
                fin := xn.findRoute(rctx, method, xsearch)
                if fin != nil {
                    return fin
                }

                // 回溯：重置参数值
                rctx.routeParams.Values = rctx.routeParams.Values[:prevlen]
                xsearch = search
            }

            rctx.routeParams.Values = append(rctx.routeParams.Values, "")

        default:
            // catch-all 节点（通配符）
            rctx.routeParams.Values = append(rctx.routeParams.Values, search)
            xn = nds[0]
            xsearch = ""
        }

        if xn == nil {
            continue
        }

        // 检查是否找到匹配的路由
        if len(xsearch) == 0 {
            if xn.isLeaf() {
                h := xn.endpoints[method]
                if h != nil && h.handler != nil {
                    rctx.routeParams.Keys = append(rctx.routeParams.Keys, h.paramKeys...)
                    return xn
                }
                
                // 处理 405 情况
                for endpoints := range xn.endpoints {
                    if endpoints == mALL || endpoints == mSTUB {
                        continue
                    }
                    rctx.methodsAllowed = append(rctx.methodsAllowed, endpoints)
                }
                rctx.methodNotAllowed = true
            }
        }

        // 递归查找下一个节点
        fin := xn.findRoute(rctx, method, xsearch)
        if fin != nil {
            return fin
        }

        // 回溯：移除参数
        if xn.typ > ntStatic {
            if len(rctx.routeParams.Values) > 0 {
                rctx.routeParams.Values = rctx.routeParams.Values[:len(rctx.routeParams.Values)-1]
            }
        }
    }

    return nil
}
```
**tree.go:401-544**

### 4.4 参数提取机制

路由匹配过程中，参数提取遵循以下机制：

1. **参数键的存储时机**：
   - 参数键在路由注册时就被解析并存储在 `endpoint.paramKeys` 中
   - 通过 `patParamKeys()` 函数解析路由模式中的参数名

2. **参数值的提取时机**：
   - 参数值在路由匹配过程中动态提取
   - 根据节点类型（参数、正则、通配符）使用不同的提取策略

3. **双阶段存储**：
   - 首先将参数值存储到 `rctx.routeParams.Values`
   - 找到匹配的叶子节点后，将参数键从 `endpoint.paramKeys` 追加到 `rctx.routeParams.Keys`
   - 最后在 `FindRoute` 中，将 `routeParams` 的内容追加到 `URLParams`

### 4.5 路由模式与参数键解析

```go
func patParamKeys(pattern string) []string {
    pat := pattern
    paramKeys := []string{}
    for {
        ptyp, paramKey, _, _, _, e := patNextSegment(pat)
        if ptyp == ntStatic {
            return paramKeys
        }
        // 检查重复参数名
        for i := 0; i < len(paramKeys); i++ {
            if paramKeys[i] == paramKey {
                panic(fmt.Sprintf("chi: routing pattern '%s' contains duplicate param key, '%s'", pattern, paramKey))
            }
        }
        paramKeys = append(paramKeys, paramKey)
        pat = pat[e:]
    }
}
```
**tree.go:755-771**

## 5. 参数注入到 http.Request

### 5.1 注入时机

路由匹配成功后，在 `routeHTTP` 方法中进行参数注入：

```go
if _, _, h := mx.tree.FindRoute(rctx, method, routePath); h != nil {
    // 将路由上下文的参数注入到 http.Request
    for i, key := range rctx.URLParams.Keys {
        value := rctx.URLParams.Values[i]
        r.SetPathValue(key, value)
    }
    r.Pattern = rctx.routePattern

    h.ServeHTTP(w, r)
    return
}
```
**mux.go:469-479**

### 5.2 注入机制

1. **Go 1.22+ 原生支持**：
   - 使用 `r.SetPathValue(key, value)` 将参数注入到 `http.Request`
   - 这是 Go 1.22 引入的新特性，允许直接在请求中存储路径参数

2. **路由模式设置**：
   - 同时设置 `r.Pattern = rctx.routePattern`
   - 这也是 Go 1.22 的特性，用于记录匹配的路由模式

3. **兼容性**：
   - Chi 同时维护了自己的 `RouteContext` 机制
   - 这确保了在 Go 1.22 之前的版本也能正常工作

## 6. chi.URLParam() 的实现原理

### 6.1 函数定义

```go
// URLParam returns the url parameter from a http.Request object.
func URLParam(r *http.Request, key string) string {
    if rctx := RouteContext(r.Context()); rctx != nil {
        return rctx.URLParam(key)
    }
    return ""
}

// URLParamFromCtx returns the url parameter from a http.Request Context.
func URLParamFromCtx(ctx context.Context, key string) string {
    if rctx := RouteContext(ctx); rctx != nil {
        return rctx.URLParam(key)
    }
    return ""
}
```
**context.go:9-23**

### 6.2 RouteContext 获取

```go
// RouteContext returns chi's routing Context object from a
// http.Request Context.
func RouteContext(ctx context.Context) *Context {
    val, _ := ctx.Value(RouteCtxKey).(*Context)
    return val
}
```
**context.go:27-30**

### 6.3 Context.URLParam 方法

```go
// URLParam returns the corresponding URL parameter value from the request
// routing context.
func (x *Context) URLParam(key string) string {
    for k := len(x.URLParams.Keys) - 1; k >= 0; k-- {
        if x.URLParams.Keys[k] == key {
            return x.URLParams.Values[k]
        }
    }
    return ""
}
```
**context.go:100-107**

### 6.4 关键设计要点

1. **反向查找**：
   - 从最后一个参数开始向前查找
   - 这确保了在有重名参数时，最内层（最新）的参数值会被返回

2. **上下文链**：
   - `URLParams` 是一个栈结构，包含所有子路由器的参数
   - 反向查找策略使得子路由器的参数可以覆盖父路由器的同名参数

3. **类型安全**：
   - 使用类型断言 `val, _ := ctx.Value(RouteCtxKey).(*Context)`
   - 如果找不到 `RouteContext`，返回空字符串

## 7. 子路由器的参数传递

### 7.1 Mount 方法中的参数处理

当使用 `Mount` 或 `Route` 挂载子路由器时，参数传递有特殊处理：

```go
func (mx *Mux) Mount(pattern string, handler http.Handler) {
    // ... 其他代码
    
    mountHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        rctx := RouteContext(r.Context())

        // 调整 URL 路径，跳过前一个子路由器的部分
        rctx.RoutePath = mx.nextRoutePath(rctx)

        // 重置连接子路由器的通配符 URLParam
        n := len(rctx.URLParams.Keys) - 1
        if n >= 0 && rctx.URLParams.Keys[n] == "*" && len(rctx.URLParams.Values) > n {
            rctx.URLParams.Values[n] = ""
        }

        handler.ServeHTTP(w, r)
    })
    
    // ... 其他代码
}
```
**mux.go:309-322**

### 7.2 路径调整机制

```go
func (mx *Mux) nextRoutePath(rctx *Context) string {
    routePath := "/"
    nx := len(rctx.routeParams.Keys) - 1 // 列表中最后一个参数的索引
    if nx >= 0 && rctx.routeParams.Keys[nx] == "*" && len(rctx.routeParams.Values) > nx {
        routePath = "/" + rctx.routeParams.Values[nx]
    }
    return routePath
}
```
**mux.go:487-494**

### 7.3 子路由器参数传递流程

1. **路径前缀匹配**：
   - 父路由器使用通配符模式 `pattern/*` 匹配路径
   - 通配符捕获剩余路径作为参数值

2. **路径调整**：
   - 子路由器通过 `nextRoutePath()` 获取通配符捕获的路径
   - 将其设置为 `rctx.RoutePath`，作为子路由器的路由起点

3. **参数重置**：
   - 重置通配符参数的值，避免影响子路由器的参数查找
   - 这是因为通配符 `*` 只是用于路径转发，不是实际的业务参数

4. **参数继承**：
   - 子路由器可以访问父路由器的 `URLParams` 中的所有参数
   - 子路由器自己匹配的参数会追加到 `URLParams` 栈中

## 8. 完整参数流转流程图

### 8.1 单路由器场景

```
请求到达
    ↓
ServeHTTP()
    ↓
从 sync.Pool 获取 RouteContext
    ↓
Reset() 重置所有状态
    ↓
将 RouteContext 注入到请求 context
    ↓
中间件链执行（Use() 注册的中间件）
    ↓
routeHTTP()
    ↓
获取 RoutePath 和 RouteMethod
    ↓
tree.FindRoute()
    ├── 重置 routePattern 和 routeParams
    ├── findRoute() 递归匹配
    │   ├── 匹配参数节点 → 提取值到 routeParams.Values
    │   ├── 找到叶子节点 → 获取 paramKeys 到 routeParams.Keys
    │   └── 返回匹配节点
    ├── 将 routeParams 追加到 URLParams
    └── 记录 routePattern 到 RoutePatterns
    ↓
匹配成功？
    │
    ├── 是 → 注入参数到 http.Request
    │        ├── 遍历 URLParams
    │        ├── r.SetPathValue(key, value)
    │        └── r.Pattern = routePattern
    │        ↓
    │     执行 handler
    │        ↓
    │     handler 中调用 chi.URLParam(r, "id")
    │        ├── 从 request context 获取 RouteContext
    │        └── 反向遍历 URLParams.Keys 查找对应值
    │
    └── 否 → 返回 404 或 405
    ↓
请求处理完成
    ↓
将 RouteContext 放回 sync.Pool
```

### 8.2 嵌套子路由器场景

```
请求到达 /api/users/123
    ↓
父路由器 ServeHTTP()
    ↓
创建并注入 RouteContext
    ↓
中间件执行
    ↓
routeHTTP()
    ↓
匹配 /api/* 模式
    ├── 通配符捕获 "users/123"
    ├── 记录到 URLParams.Keys = ["*"], Values = ["users/123"]
    └── 执行 mountHandler
    ↓
mountHandler 中：
    ├── rctx.RoutePath = "/users/123" (从通配符值获取)
    ├── 重置通配符值：URLParams.Values[0] = ""
    └── 调用子路由器 ServeHTTP()
    ↓
子路由器 ServeHTTP()
    ├── 检查到已有 RouteContext，直接使用
    └── 执行子路由器的中间件链
    ↓
子路由器 routeHTTP()
    ├── 使用 rctx.RoutePath = "/users/123" 进行匹配
    ├── 匹配 /users/{id} 模式
    │   ├── 提取 "123" 到 routeParams.Values
    │   ├── 获取 paramKeys = ["id"]
    │   └── 追加到 URLParams: Keys = ["*", "id"], Values = ["", "123"]
    └── 执行 handler
    ↓
handler 中调用 chi.URLParam(r, "id")
    ├── 获取 RouteContext
    ├── 反向遍历 URLParams.Keys
    ├── 找到 "id" 对应的值 "123"
    └── 返回 "123"
```

## 9. 关键设计特性

### 9.1 对象池复用

1. **性能优化**：
   - 使用 `sync.Pool` 复用 `RouteContext` 对象
   - 减少内存分配和 GC 压力

2. **状态管理**：
   - 每次使用前调用 `Reset()` 重置状态
   - 确保对象状态干净，避免数据污染

### 9.2 栈式参数存储

1. **双存储设计**：
   - `routeParams`：当前路由器的参数（临时）
   - `URLParams`：跨路由器的参数栈（持久）

2. **嵌套支持**：
   - 每个子路由器的参数追加到栈中
   - 反向查找确保内层参数优先

### 9.3 通配符特殊处理

1. **路径转发**：
   - 通配符 `*` 主要用于子路由器路径转发
   - 捕获的值作为子路由器的 `RoutePath`

2. **参数重置**：
   - 转发后重置通配符参数值
   - 避免影响业务参数的查找

### 9.4 与 Go 1.22+ 集成

1. **双轨制**：
   - 同时支持 Chi 自己的 `RouteContext` 和 Go 1.22 的 `SetPathValue`
   - 确保向后兼容

2. **参数同步**：
   - 路由匹配后，将 `URLParams` 同步到 `http.Request`
   - 用户可以通过 `r.PathValue()` 或 `chi.URLParam()` 两种方式获取

## 10. 实际使用示例

### 10.1 基本参数获取

```go
r := chi.NewRouter()

// 定义带参数的路由
r.Get("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    // 使用 chi.URLParam 获取参数
    userID := chi.URLParam(r, "id")
    
    // 或者使用 Go 1.22+ 的 r.PathValue
    // userID := r.PathValue("id")
    
    w.Write([]byte("User ID: " + userID))
})
```

### 10.2 正则表达式参数

```go
r.Get("/articles/{id:\\d+}", func(w http.ResponseWriter, r *http.Request) {
    articleID := chi.URLParam(r, "id")
    // 只有数字 ID 会匹配到此路由
})
```

### 10.3 通配符路由

```go
r.Get("/files/*", func(w http.ResponseWriter, r *http.Request) {
    filePath := chi.URLParam(r, "*")
    // filePath 包含通配符匹配的所有内容
})
```

### 10.4 嵌套路由参数

```go
r.Route("/api", func(r chi.Router) {
    r.Route("/users/{userID}", func(r chi.Router) {
        r.Get("/profile", func(w http.ResponseWriter, r *http.Request) {
            // 可以获取父路由器的参数
            userID := chi.URLParam(r, "userID")
            w.Write([]byte("User Profile: " + userID))
        })
    })
})
```

## 11. 总结

Chi 框架的路由上下文与参数传递机制是一个精心设计的系统，具有以下核心特点：

1. **高效的对象复用**：
   - 使用 `sync.Pool` 管理 `RouteContext` 对象
   - 通过 `Reset()` 方法实现状态重置和对象复用

2. **灵活的参数存储**：
   - 采用双栈结构（`routeParams` 和 `URLParams`）
   - 支持多层嵌套路由器的参数传递
   - 反向查找策略确保内层参数优先

3. **清晰的生命周期**：
   - `RouteContext` 在请求入口创建
   - 路由匹配过程中提取和存储参数
   - 匹配成功后注入到 `http.Request`
   - 请求处理完成后放回对象池

4. **良好的兼容性**：
   - 同时支持 Chi 自己的 `URLParam()` 和 Go 1.22+ 的 `PathValue()`
   - 子路由器参数传递机制设计巧妙

5. **强大的路由匹配**：
   - 支持静态路径、命名参数、正则表达式、通配符
   - 参数键在路由注册时解析，参数值在匹配时提取
   - 递归匹配和回溯机制确保正确的路由选择

理解这套机制有助于开发者：
- 正确使用路由参数
- 理解参数在中间件和 handler 中的传递过程
- 排查参数相关的问题
- 充分利用 Chi 框架的路由能力

## 12. 参考源码位置

- `context.go`：`Context` 结构体、`URLParam` 函数、`RouteContext` 函数
- `mux.go`：`ServeHTTP` 方法、`routeHTTP` 方法、`Mount` 方法
- `tree.go`：`FindRoute` 方法、`findRoute` 方法、`patParamKeys` 函数
