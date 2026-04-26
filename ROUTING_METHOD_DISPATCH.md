# go-chi 路由机制深度分析（方法分发篇）

## 方法分发、405 响应生成与 inline 路由机制详解

---

## 1. 概述

在前两篇报告中，我们分析了 Radix Tree 的节点存储结构、匹配顺序和回溯机制。本文将深入分析路由命中后的方法分发层，包括：

1. **`endpoints` 字段**：如何把 HTTP 方法映射到具体 handler
2. **`methodsAllowed` 收集**：路径匹配但方法不支持时，如何收集允许的方法并生成 405 响应
3. **inline 路由**：通过 `With()`、`Group()` 创建的内联子路由与普通路由在 endpoint 层的存储和处理差异

---

## 2. 方法类型与位掩码设计

### 2.1 methodTyp 类型定义

chi 使用**位掩码（bitmask）来表示 HTTP 方法，这是一种高效的设计：

```go
type methodTyp uint

const (
    mSTUB methodTyp = 1 << iota  // = 1  (0b0000000001)
    mCONNECT                       // = 2  (0b0000000010)
    mDELETE                      // = 4  (0b0000000100)
    mGET                         // = 8  (0b0000001000)
    mHEAD                        // = 16 (0b0000010000)
    mOPTIONS                     // = 32 (0b0000100000)
    mPATCH                       // = 64 (0b0001000000)
    mPOST                        // = 128 (0b0010000000)
    mPUT                         // = 256 (0b0100000000)
    mTRACE                       // = 512 (0b1000000000)
)

// 所有标准方法的组合
var mALL = mCONNECT | mDELETE | mGET | mHEAD |
    mOPTIONS | mPATCH | mPOST | mPUT | mTRACE
```
**[tree.go:17-33](tree.go#L17-L33)**

### 2.2 方法映射表

```go
var methodMap = map[string]methodTyp{
    http.MethodConnect: mCONNECT,
    http.MethodDelete:  mDELETE,
    http.MethodGet:     mGET,
    http.MethodHead:    mHEAD,
    http.MethodOptions: mOPTIONS,
    http.MethodPatch:   mPATCH,
    http.MethodPost:    mPOST,
    http.MethodPut:     mPUT,
    http.MethodTrace:   mTRACE,
}

var reverseMethodMap = map[methodTyp]string{
    mCONNECT: http.MethodConnect,
    mDELETE:  http.MethodDelete,
    mGET:     http.MethodGet,
    mHEAD:    http.MethodHead,
    mOPTIONS: http.MethodOptions,
    mPATCH:   http.MethodPatch,
    mPOST:    http.MethodPost,
    mPUT:     http.MethodPut,
    mTRACE:   http.MethodTrace,
}
```
**[tree.go:35-57](tree.go#L35-L57)**

### 2.3 位掩码设计的优势

1. **高效的方法组合**：
   - `mALL` 是所有方法的按位或，可以一次性表示"所有方法"
   - `mALL | mSTUB` 可以表示"所有方法 + STUB 标记"

2. **高效的位运算**：
   - `method & mSTUB == mSTUB` 检查是否包含 STUB 标记
   - `method & mALL == mALL` 检查是否是"所有方法"

3. **可扩展性**：
   - 支持通过 `RegisterMethod()` 添加自定义方法
   - 每个新方法分配一个新的 bit 位

---

## 3. endpoints 字段结构与方法映射

### 3.1 数据结构定义

```go
// endpoints 是 HTTP 方法常量到 handlers 的映射
type endpoints map[methodTyp]*endpoint

type endpoint struct {
    handler http.Handler  // 具体的 handler
    pattern string        // 路由模式（如 "/articles/{id}"）
    paramKeys []string    // 参数名列表（如 ["id"]）
}
```
**[tree.go:115-128](tree.go#L115-L128)**

### 3.2 Value 方法：懒加载创建 endpoint

```go
func (s endpoints) Value(method methodTyp) *endpoint {
    mh, ok := s[method]
    if !ok {
        // 如果方法不存在，创建一个新的空 endpoint
        mh = &endpoint{}
        s[method] = mh
    }
    return mh
}
```
**[tree.go:130-137](tree.go#L130-L137)**

**设计意图**：
- 避免在访问时才创建 endpoint，节省内存
- 确保 `s[method]` 永远不会返回 nil

### 3.3 setEndpoint：设置叶子节点的 handler

这是路由注册时设置 endpoint 的核心方法：

```go
func (n *node) setEndpoint(method methodTyp, handler http.Handler, pattern string) {
    // 1. 懒初始化 endpoints map
    if n.endpoints == nil {
        n.endpoints = make(endpoints)
    }

    // 2. 从路由模式中提取参数名
    paramKeys := patParamKeys(pattern)

    // 3. 处理特殊方法类型
    if method&mSTUB == mSTUB {
        // STUB 方法：只设置 handler，不设置 pattern 和 paramKeys
        n.endpoints.Value(mSTUB).handler = handler
    }
    
    if method&mALL == mALL {
        // 所有方法（Handle/HandleFunc）
        // 先设置 mALL 的 endpoint
        h := n.endpoints.Value(mALL)
        h.handler = handler
        h.pattern = pattern
        h.paramKeys = paramKeys
        
        // 再为每个具体方法设置相同的 endpoint
        for _, m := range methodMap {
            h := n.endpoints.Value(m)
            h.handler = handler
            h.pattern = pattern
            h.paramKeys = paramKeys
        }
    } else {
        // 单个具体方法（Get/Post/Put/Delete 等）
        h := n.endpoints.Value(method)
        h.handler = handler
        h.pattern = pattern
        h.paramKeys = paramKeys
    }
}
```
**[tree.go:344-372](tree.go#L344-L372)**

### 3.4 方法注册示例

假设有以下路由注册：

```go
r.Get("/articles", listHandler)        // method = mGET
r.Post("/articles", createHandler)    // method = mPOST
r.Handle("/articles/{id}", detailHandler) // method = mALL
```

注册后的 `endpoints` 结构：

```
/articles 节点的 endpoints:
{
    mGET:  {handler: listHandler, pattern: "/articles", paramKeys: []},
    mPOST: {handler: createHandler, pattern: "/articles", paramKeys: []},
}

/articles/{id} 节点的 endpoints:
{
    mALL:     {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mCONNECT:  {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mDELETE:   {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mGET:      {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mHEAD:     {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mOPTIONS:  {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mPATCH:    {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mPOST:     {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mPUT:      {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
    mTRACE:    {handler: detailHandler, pattern: "/articles/{id}", paramKeys: ["id"]},
}
```

### 3.5 mSTUB 标记的作用

`mSTUB` 是一个特殊的方法标记，主要用于：

1. **Mount 时的内部标记**：
   ```go
   // 在 Mount 中
   method := mALL
   subroutes, _ := handler.(Routes)
   if subroutes != nil {
       method |= mSTUB  // 添加 STUB 标记
   }
   ```
   **[mux.go:330-334](mux.go#L330-L334)**

2. **Walk 遍历**：用于区分普通路由和 Mount 路由

---

## 4. methodsAllowed 收集与 405 响应生成

### 4.1 问题场景

当路由**路径匹配成功**但**HTTP 方法不支持**时，chi 会：
1. 收集该路径支持的所有 HTTP 方法
2. 设置 `methodNotAllowed = true`
3. 最终返回 `405 Method Not Allowed` 响应，并在 `Allow` 头中列出支持的方法

### 4.2 Context 中的相关字段

```go
type Context struct {
    // ...
    methodsAllowed   []methodTyp  // 允许的方法列表
    methodNotAllowed bool          // 是否是 405 标记
    // ...
}
```
**[context.go:77-78](context.go#L77-L78)**

### 4.3 findRoute 中的收集逻辑

在 `findRoute` 中有**两处**收集 `methodsAllowed` 的逻辑：

#### 第一处：参数/正则节点循环内部

```go
case ntParam, ntRegexp:
    // ... 节点匹配逻辑...
    
    if len(xsearch) == 0 {
        if xn.isLeaf() {
            h := xn.endpoints[method]
            if h != nil && h.handler != nil {
                // 方法匹配成功，返回节点
                rctx.routeParams.Keys = append(rctx.routeParams.Keys, h.paramKeys...)
                return xn
            }
            
            // ========== 关键点：方法不匹配，收集允许的方法 ==========
            for endpoints := range xn.endpoints {
                // 跳过 mALL 和 mSTUB，只收集具体方法
                if endpoints == mALL || endpoints == mSTUB {
                    continue
                }
                rctx.methodsAllowed = append(rctx.methodsAllowed, endpoints)
            }
            
            // 设置 405 标记
            rctx.methodNotAllowed = true
        }
    }
```
**[tree.go:461-479](tree.go#L461-L479)**

#### 第二处：所有节点类型的统一处理

```go
// 在 switch 语句之后，对所有类型的节点统一处理：

if xn == nil {
    continue
}

// 检查是否找到完整匹配
if len(xsearch) == 0 {
    if xn.isLeaf() {
        h := xn.endpoints[method]
        if h != nil && h.handler != nil {
            // 方法匹配成功
            rctx.routeParams.Keys = append(rctx.routeParams.Keys, h.paramKeys...)
            return xn
        }
        
        // ========== 关键点：方法不匹配，收集允许的方法 ==========
        for endpoints := range xn.endpoints {
            if endpoints == mALL || endpoints == mSTUB {
                continue
            }
            rctx.methodsAllowed = append(rctx.methodsAllowed, endpoints)
        }
        
        rctx.methodNotAllowed = true
    }
}
```
**[tree.go:506-526](tree.go#L506-L526)**

### 4.4 为什么跳过 mALL 和 mSTUB？

```go
for endpoints := range xn.endpoints {
    if endpoints == mALL || endpoints == mSTUB {
        continue
    }
    rctx.methodsAllowed = append(rctx.methodsAllowed, endpoints)
}
```

**原因**：
1. **`mALL`** 是所有方法的组合位掩码，不是具体的 HTTP 方法
   - 如果路由用 `Handle()` 注册（`mALL`），`endpoints 中会同时存在：
     - `mALL` 键
     - 以及所有具体方法键（`mGET`、`mPOST` 等）
   - 收集时应该收集具体方法，跳过 `mALL`

2. **`mSTUB`** 是内部标记，不是 HTTP 方法
   - 用于 Mount 路由的内部标识
   - 不应该出现在 `Allow` 响应头中

### 4.5 完整的 405 响应生成

在 routeHTTP 中的最终处理

```go
func (mx *Mux) routeHTTP(w http.ResponseWriter, r *http.Request) {
    // ... 路由匹配...
    
    // FindRoute 查找路由
    if _, _, h := mx.tree.FindRoute(rctx, method, routePath); h != nil {
        // 匹配成功，执行 handler
        h.ServeHTTP(w, r)
        return
    }
    
    // ========== 关键点：405 或 404 ==========
    if rctx.methodNotAllowed {
        // 路径匹配但方法不支持 → 405
        mx.MethodNotAllowedHandler(rctx.methodsAllowed...).ServeHTTP(w, r)
    } else {
        // 路径不匹配 → 404
        mx.NotFoundHandler().ServeHTTP(w, r)
    }
}
```
**[mux.go:468-484](mux.go#L468-L484)**

### 4.6 MethodNotAllowedHandler 实现

```go
func (mx *Mux) MethodNotAllowedHandler(methodsAllowed ...methodTyp) http.HandlerFunc {
    if mx.methodNotAllowedHandler != nil {
        return mx.methodNotAllowedHandler
    }
    return methodNotAllowedHandler(methodsAllowed...)
}

// 辅助函数：生成 405 响应
func methodNotAllowedHandler(methodsAllowed ...methodTyp) func(w http.ResponseWriter, r *http.Request) {
    return func(w http.ResponseWriter, r *http.Request) {
        // 在 Allow 头中列出所有允许的方法
        for _, m := range methodsAllowed {
            w.Header().Add("Allow", reverseMethodMap[m])
        }
        w.WriteHeader(405)
        w.Write(nil)  // 空响应体
    }
}
```
**[mux.go:407-426](mux.go#L407-L426)**

### 4.7 完整示例

假设有以下路由：

```go
r := chi.NewRouter()
r.Get("/articles", listHandler)    // GET /articles
r.Post("/articles", createHandler)   // POST /articles
```

请求 `PUT /articles`：

```
匹配流程：
1. findRoute 匹配到 "/articles" 节点（路径匹配成功）
2. 检查 endpoints[mPUT].handler → nil（方法不匹配）
3. 遍历 endpoints，收集：
   - mGET → 添加到 methodsAllowed
   - mPOST → 添加到 methodsAllowed
4. 设置 methodNotAllowed = true
5. findRoute 返回 nil
6. routeHTTP 检测到 methodNotAllowed = true
7. 调用 MethodNotAllowedHandler([mGET, mPOST])
8. 响应：
   - Status: 405 Method Not Allowed
   - Headers: Allow: GET, Allow: POST
```

---

## 5. inline 路由机制深度解析

### 5.1 什么是 inline 路由？

inline 路由是通过以下方式创建的路由：

```go
// 方式1：With() 创建带内联中间件的路由
r.With(middleware1, middleware2).Get("/protected", protectedHandler)

// 方式2：Group() 创建路由组
r.Group(func(r chi.Router) {
    r.Use(middleware1)
    r.Get("/group1", handler1)
    r.Get("/group2", handler2)
})

// 方式3：Route() 创建子路由（注意：Route 内部调用 Mount，不是真正的 inline）
r.Route("/api", func(r chi.Router) {
    r.Get("/users", listUsers)  // 这是 Mount 的子路由，不是 inline
})
```

### 5.2 Mux 中的 inline 字段

```go
type Mux struct {
    // ...
    inline bool  // 是否是 inline Mux
    parent *Mux // 父 Mux 引用
    // ...
}
```
**[mux.go:47-48](mux.go#L47-L48)**

### 5.3 With() 方法实现

```go
func (mx *Mux) With(middlewares ...func(http.Handler) http.Handler) Router {
    // ========== 关键点1：如果是非 inline 且 handler 未构建，先构建 ==========
    if !mx.inline && mx.handler == nil {
        mx.updateRouteHandler()
    }

    // ========== 关键点2：复制中间件 ==========
    var mws Middlewares
    if mx.inline {
        // 如果已经是 inline，复制父 Mux 的中间件
        mws = make(Middlewares, len(mx.middlewares))
        copy(mws, mx.middlewares)
    }
    // 添加新的中间件
    mws = append(mws, middlewares...)

    // ========== 关键点3：创建新的 inline Mux ==========
    im := &Mux{
        pool: mx.pool,
        inline: true,           // 标记为 inline
        parent: mx,              // 指向父 Mux
        tree: mx.tree,           // 共享同一棵 Radix Tree！
        middlewares: mws,         // 合并后的中间件
        notFoundHandler: mx.notFoundHandler,
        methodNotAllowedHandler: mx.methodNotAllowedHandler,
    }

    return im
}
```
**[mux.go:236-257](mux.go#L236-L257)**

**关键设计**：
- **共享 tree**：inline Mux 和父 Mux 共享同一棵 Radix Tree
- **独立 middlewares**：inline Mux 有自己的中间件链（父 + 新）
- **inline = true**：标记为内联模式

### 5.4 Group() 方法实现

```go
func (mx *Mux) Group(fn func(r Router)) Router {
    // With() 创建一个空中间件的 inline Mux
    im := mx.With()
    if fn != nil {
        fn(im)  // 在 inline Mux 上注册路由
    }
    return im
}
```
**[mux.go:262-268](mux.go#L262-L268)**

### 5.5 handle() 方法中的 inline 处理

这是 inline 路由最关键的区别：

```go
func (mx *Mux) handle(method methodTyp, pattern string, handler http.Handler) *node {
    // ...
    
    // ========== 关键点：inline Mux 的特殊处理 ==========
    var h http.Handler
    if mx.inline {
        // ========== inline Mux：包装 handler ==========
        // 1. 设置 handler 为 routeHTTP（用于子路由递归）
        mx.handler = http.HandlerFunc(mx.routeHTTP)
        
        // 2. 用 Chain 包装 handler：中间件链 + 原始 handler
        h = Chain(mx.middlewares...).Handler(handler)
    } else {
        // ========== 普通 Mux：直接使用 handler
        h = handler
    }

    // 3. 插入到共享的 Radix Tree
    return mx.tree.InsertRoute(method, pattern, h)
}
```
**[mux.go:416-437](mux.go#L416-L437)**

### 5.6 Chain 包装机制

```go
// ChainHandler 是支持 handler 组合
type ChainHandler struct {
    Endpoint    http.Handler      // 原始的 endpoint handler
    chain       http.Handler       // 包装后的 handler（中间件链）
    Middlewares Middlewares          // 中间件列表
}

func (c *ChainHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    c.chain.ServeHTTP(w, r)  // 执行包装后的 handler
}

// chain 函数：构建中间件链
func chain(middlewares []func(http.Handler) http.Handler, endpoint http.Handler) http.Handler {
    if len(middlewares) == 0 {
        return endpoint
    }
    
    // 从后往前包装
    h := middlewares[len(middlewares)-1](endpoint)
    for i := len(middlewares) - 2; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}
```
**[chain.go:22-49](chain.go#L22-L49)**

### 5.7 inline 路由 vs 普通路由 vs Mount 子路由

| 特性 | 普通路由 | inline 路由 (With/Group) | Mount 子路由 (Route) |
|------|----------|---------------------------|----------------------|
| **tree 共享** | 自己的 tree | 与父 Mux 共享 tree | 独立的 tree |
| **中间件处理** | 无（或使用 Use() 的全局中间件 | 注册时用 Chain 包装 | 通过 mountHandler 转接 |
| **handler 存储** | 原始 handler | ChainHandler（包装了中间件） | mountHandler（转接 |
| **路由模式** | 完整路径 | 完整路径（共享 tree） | 前缀 + 子路径（独立 tree） |
| **参数可见性** | 自己的参数 | 自己的参数 | 父参数 + 子参数 |

### 5.8 inline 路由示例

假设有以下代码：

```go
r := chi.NewRouter()
r.Use(globalMiddleware)  // 全局中间件

// 普通路由
r.Get("/public", publicHandler)

// inline 路由（With）
r.With(authMiddleware).Get("/protected", protectedHandler)

// inline 路由（Group）
r.Group(func(r chi.Router) {
    r.Use(logMiddleware)
    r.Get("/group1", handler1)
    r.Get("/group2", handler2)
})
```

**注册后的 tree 结构**：

```
Radix Tree（所有路由共享同一棵树）：
│
├── /public
│   └── endpoints: {mGET: {handler: publicHandler, ...}}
│
├── /protected
│   └── endpoints: {mGET: {handler: ChainHandler{
│                           Endpoint: protectedHandler,
│                           chain: chain([globalMiddleware, authMiddleware], protectedHandler)
│                        }, ...}}
│
├── /group1
│   └── endpoints: {mGET: {handler: ChainHandler{
│                           Endpoint: handler1,
│                           chain: chain([globalMiddleware, logMiddleware], handler1)
│                        }, ...}}
│
└── /group2
    └── endpoints: {mGET: {handler: ChainHandler{
                            Endpoint: handler2,
                            chain: chain([globalMiddleware, logMiddleware], handler2)
                         }, ...}}
```

**请求匹配流程**：

请求 `GET /protected`：
```
1. 匹配到 /protected 节点
2. 获取 endpoints[mGET].handler → ChainHandler
3. 执行 ChainHandler.ServeHTTP()
   → 执行 chain（先 globalMiddleware，再 authMiddleware，最后 protectedHandler）
```

### 5.9 inline Mux 的 handler 设置

```go
if mx.inline {
    mx.handler = http.HandlerFunc(mx.routeHTTP)
    // ...
}
```

**为什么要设置 `mx.handler = mx.routeHTTP`？**

这是为了支持**嵌套的 With() 调用：

```go
// 嵌套 With
r.With(mw1).With(mw2).Get("/deep", handler)
```

当第二个 `With()` 调用时：
```go
func (mx *Mux) With(...) {
    if !mx.inline && mx.handler == nil {
        mx.updateRouteHandler()  // 第一个 With 会触发
    }
    // ...
}
```

对于已经是 inline 的 Mux，这个条件不触发。但 `mx.handler = mx.routeHTTP` 的设置是在 `handle()` 中：

```go
func (mx *Mux) handle(...) {
    if mx.inline {
        mx.handler = http.HandlerFunc(mx.routeHTTP)
        // ...
    }
}
```

这个设置的实际作用是**允许 inline Mux 作为独立的 handler 被调用**，但实际上由于 inline Mux 共享父 Mux 的 tree，这个设置在正常使用中很少被触发。

---

## 6. 完整的方法分发流程图

### 6.1 路由注册时的方法设置

```
┌─────────────────────────────────────────────────────────────────────┐
│                      路由注册流程                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  r.Get("/path", handler)                                            │
│       │                                                              │
│       ▼                                                              │
│  Mux.handle(method=mGET, pattern="/path", handler)                 │
│       │                                                              │
│       ├─ inline? ──Yes──► h = Chain(middlewares).Handler(handler) │
│       │                     │                                       │
│       │                     ▼                     │                               │
│       │                ChainHandler{                                    │
│       │                ├── Endpoint: handler                                   │
│       │                ├── chain: 中间件链包装的 handler          │
│       │                └── Middlewares: 中间件列表               │
│       │                                                              │
│       └─ No ──► h = handler（原始）                                  │
│       │                                                              │
│       ▼                                                              │
│  tree.InsertRoute(mGET, "/path", h)                                  │
│       │                                                              │
│       ▼                                                              │
│  node.setEndpoint(mGET, h, "/path")                                 │
│       │                                                              │
│       └─► endpoints[mGET] = &endpoint{                            │
│              handler: h,                                              │
│              pattern: "/path",                                         │
│              paramKeys: [...]                                          │
│          }                                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 请求匹配时的方法分发

```
┌─────────────────────────────────────────────────────────────────────┐
│                      请求匹配流程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  请求: PUT /articles                                                │
│       │                                                              │
│       ▼                                                              │
│  Mux.routeHTTP()                                                      │
│       │                                                              │
│       ▼                                                              │
│  tree.FindRoute(rctx, mPUT, "/articles")                             │
│       │                                                              │
│       ▼                                                              │
│  findRoute() 递归匹配                                               │
│       │                                                              │
│       ├─ 路径匹配成功 ✓                                                │
│       │       │                                                      │
│       │       ▼                                                      │
│       │  xn.isLeaf() = true                                          │
│       │       │                                                      │
│       │       ▼                                                      │
│       │  h = xn.endpoints[mPUT]                                    │
│       │       │                                                      │
│       │       ├─ h != nil && h.handler != nil ──► 方法匹配成功          │
│       │       │                               │                       │
│       │       │                               ▼                       │
│       │       │                    return xn                                  │
│       │       │                               │                       │
│       │       │                               ▼                       │
│       │       │                    routeHTTP:                       │
│       │       │                      ├─ 设置 r.SetPathValue()           │
│       │       │                      ├─ r.Pattern = pattern       │
│       │       │                      └─ h.ServeHTTP() 执行 handler     │
│       │       │                                                         │
│       │       │                                                         │
│       │       └─ h == nil || h.handler == nil ──► 方法不匹配        │
│       │                                       │                         │
│       │                                       ▼                         │
│       │                              遍历 endpoints:                    │
│       │                                for m := range xn.endpoints {│
│       │                                    if m == mALL \|\| m == mSTUB │
│       │                                        continue              │
│       │                                    methodsAllowed = append(...) │
│       │                                }                             │
│       │                                       │                         │
│       │                                       ▼                         │
│       │                              methodNotAllowed = true            │
│       │                                       │                         │
│       │                                       ▼                         │
│       │                              return nil                      │
│       │                                                              │
│       └─ 路径匹配失败 ✗                                               │
│               │                                                      │
│               ▼                                                      │
│          return nil                                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────┐
│  routeHTTP 后续处理                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  if h != nil {                                                     │
│      h.ServeHTTP(w, r)  // 执行 handler                             │
│      return                                                          │
│  }                                                                   │
│                                                                      │
│  if rctx.methodNotAllowed {                                         │
│      // 405 Method Not Allowed                                      │
│      mx.MethodNotAllowedHandler(rctx.methodsAllowed...).ServeHTTP()  │
│      // 响应:                                                          │
│      //   Status: 405                                                 │
│      //   Allow: GET, POST, ...                                         │
│  } else {                                                            │
│      // 404 Not Found                                                 │
│      mx.NotFoundHandler().ServeHTTP()                                 │
│  }                                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 关键设计洞察

### 7.1 位掩码方法设计的优势

1. **高效的方法组合**：`mALL` 可以一次性表示所有方法
2. **高效的位运算检查**：`method & mSTUB == mSTUB` 检查标记
3. **可扩展性**：支持 `RegisterMethod()` 添加自定义方法

### 7.2 endpoints 存储的优化

1. **懒加载**：`Value()` 方法在访问时才创建 endpoint
2. **mALL 展开**：`Handle()` 注册时会展开为所有具体方法
   - 这样 `findRoute` 可以直接用具体方法查找，不需要额外处理 `mALL`

### 7.3 405 收集的精确性

1. **两处收集**：参数节点循环内部 + switch 后统一处理
   - 确保所有可能的路径匹配都能收集到允许的方法
2. **跳过 mALL 和 mSTUB**：只收集具体的 HTTP 方法
3. **methodNotAllowed 标记**：区分"路径匹配但方法不支持" vs "路径完全不匹配"

### 7.4 inline 路由的巧妙设计

1. **共享 tree**：inline Mux 和父 Mux 共享同一棵 Radix Tree
   - 所有路由在同一棵树中，匹配效率高
2. **Chain 包装**：注册时用中间件链包装 handler
   - 中间件在路由注册时就确定，匹配时直接执行
3. **独立的 middlewares**：每个 inline Mux 有自己的中间件链
   - 支持灵活的中间件组合

### 7.5 inline vs Mount 的本质区别

| 维度 | inline (With/Group) | Mount (Route) |
|------|---------------------|---------------|
| **Tree** | 共享 | 独立 |
| **中间件** | 注册时包装 | 运行时转接 |
| **适用场景** | 同一组路由的不同中间件组合 | 模块化的子系统 |
| **参数隔离** | 无（共享路径） | 有（父路径参数 + 子路径参数） |

---

## 8. 总结

### 8.1 方法分发核心要点

| 要点 | 说明 |
|------|------|
| **方法表示** | 使用位掩码 `methodTyp`，支持位运算组合 |
| **endpoints** | `map[methodTyp]*endpoint`，存储方法到 handler 的映射 |
| **setEndpoint** | 处理 `mALL` 时展开为所有具体方法 |
| **Value 方法** | 懒加载创建 endpoint，避免 nil panic |

### 8.2 405 响应生成核心要点

| 要点 | 说明 |
|------|------|
| **收集时机** | 路径匹配成功但方法不匹配时 |
| **收集位置** | `findRoute` 中有两处收集逻辑 |
| **跳过项** | `mALL`（方法组合）和 `mSTUB`（内部标记） |
| **响应生成** | `methodNotAllowedHandler` 设置 `Allow` 头和 405 状态 |

### 8.3 inline 路由核心要点

| 要点 | 说明 |
|------|------|
| **创建方式** | `With()` 和 `Group()` |
| **核心特性** | 共享父 Mux 的 Radix Tree |
| **中间件处理** | 注册时用 `Chain()` 包装 handler 为 `ChainHandler` |
| **与 Mount 区别** | inline 共享 tree，Mount 有独立 tree |

### 8.4 完整的数据流向

```
路由注册:
  代码 → Mux.handle() → Chain 包装(inline) → tree.InsertRoute() → node.setEndpoint() → endpoints[method]

请求匹配:
  请求 → Mux.routeHTTP() → tree.FindRoute() → findRoute() 递归
           │
           ├─ 方法匹配 → 获取 endpoints[method].handler → 执行 ChainHandler → 中间件链 → endpoint
           │
           └─ 方法不匹配 → 收集 methodsAllowed → methodNotAllowed = true → 405 响应
```

chi 的方法分发机制设计精巧，通过位掩码、懒加载、Chain 包装等技术，实现了高性能、高灵活性的 HTTP 路由功能。理解这些机制有助于更好地使用 chi 进行 Web 开发，并能在遇到复杂路由场景时进行有效的调试。
