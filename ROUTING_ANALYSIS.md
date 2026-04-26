# go-chi 路由机制深度分析

## 1. 概述

go-chi 的路由核心基于 **Radix Tree（基数树/前缀树）** 实现，这是一种高效的字符串匹配数据结构。相比传统的 trie 树，Radix Tree 通过路径压缩减少了节点数量，提升了内存效率和查找速度。

## 2. 节点类型与存储结构

### 2.1 节点类型 (nodeTyp)

chi 定义了四种节点类型，用于区分不同的路由模式：

```go
type nodeTyp uint8

const (
    ntStatic   nodeTyp = iota // 静态路由，如 /home
    ntRegexp                   // 正则表达式路由，如 /{id:[0-9]+}
    ntParam                    // 参数路由，如 /{user}
    ntCatchAll                 // 通配符路由，如 /api/v1/*
)
```
**[tree.go:79-86](tree.go#L79-L86)**

### 2.2 节点结构 (node)

```go
type node struct {
    subroutes Routes              // 叶子节点上的子路由
    rex *regexp.Regexp            // 正则表达式节点的匹配器
    endpoints endpoints            // HTTP 方法到 handler 的映射
    prefix string                  // 节点存储的公共前缀
    children [ntCatchAll + 1]nodes // 核心：按类型分组的子节点数组
    tail byte                      // 子节点前缀的尾部分隔符
    typ nodeTyp                    // 当前节点类型
    label byte                     // 前缀的第一个字节
}
```
**[tree.go:88-113](tree.go#L88-L113)**

### 2.3 核心设计：按类型分组的子节点数组

最关键的设计是 `children` 字段：

```go
children [ntCatchAll + 1]nodes
```

这是一个**固定大小的数组**，大小为 `4`（因为 `ntCatchAll = 3`），每个索引对应一种节点类型：

| 索引 | 节点类型 | 说明 |
|------|----------|------|
| `children[0]` | `ntStatic` | 静态节点列表 |
| `children[1]` | `ntRegexp` | 正则表达式节点列表 |
| `children[2]` | `ntParam` | 参数节点列表 |
| `children[3]` | `ntCatchAll` | 通配符节点列表 |

**这种设计的优势：**
1. **类型隔离**：不同类型的路由节点物理上分开存储
2. **匹配顺序可控**：遍历数组的顺序决定了匹配优先级
3. **查找高效**：静态节点使用二分查找，参数节点顺序遍历

### 2.4 节点标签和尾部分隔符

每个节点有两个重要的字节标识：
- **`label`**：前缀的第一个字节，用于快速定位节点
- **`tail`**：对于参数/正则节点，标识参数段的结束分隔符（如 `/`、`:`、`!` 等）

例如路由 `/{id}:delete`：
- 参数节点的 `label = '{'`
- 参数节点的 `tail = ':'`

这使得 chi 支持非 `/` 分隔的参数，如 `/{id}.json`、`/{id}:edit` 等。

## 3. 路由注册流程

### 3.1 入口：Mux.handle()

路由注册的入口在 `mux.go`：

```go
func (mx *Mux) handle(method methodTyp, pattern string, handler http.Handler) *node {
    // 1. 验证路由模式必须以 '/' 开头
    if len(pattern) == 0 || pattern[0] != '/' {
        panic(fmt.Sprintf("chi: routing pattern must begin with '/' in '%s'", pattern))
    }

    // 2. 构建中间件链（如果尚未构建）
    if !mx.inline && mx.handler == nil {
        mx.updateRouteHandler()
    }

    // 3. 处理 inline mux 的中间包装
    var h http.Handler
    if mx.inline {
        mx.handler = http.HandlerFunc(mx.routeHTTP)
        h = Chain(mx.middlewares...).Handler(handler)
    } else {
        h = handler
    }

    // 4. 插入到 Radix Tree
    return mx.tree.InsertRoute(method, pattern, h)
}
```
**[mux.go:416-437](mux.go#L416-L437)**

### 3.2 核心：node.InsertRoute()

`InsertRoute` 是路由插入的核心实现，采用循环方式遍历树：

```go
func (n *node) InsertRoute(method methodTyp, pattern string, handler http.Handler) *node {
    var parent *node
    search := pattern

    for {
        // 1. 路径耗尽，设置当前节点为叶子节点
        if len(search) == 0 {
            n.setEndpoint(method, handler, pattern)
            return n
        }

        // 2. 解析下一段的类型（静态/参数/正则/通配符）
        var label = search[0]
        var segTail byte
        var segEndIdx int
        var segTyp nodeTyp
        var segRexpat string
        if label == '{' || label == '*' {
            segTyp, _, segRexpat, segTail, _, segEndIdx = patNextSegment(search)
        }

        // 3. 查找匹配的子节点
        parent = n
        n = n.getEdge(segTyp, label, segTail, prefix)

        // 4. 没有匹配的边，创建新节点
        if n == nil {
            child := &node{label: label, tail: segTail, prefix: search}
            hn := parent.addChild(child, search)
            hn.setEndpoint(method, handler, pattern)
            return hn
        }

        // 5. 找到参数/正则/通配符节点，跳过参数段继续
        if n.typ > ntStatic {
            search = search[segEndIdx:]
            continue
        }

        // 6. 静态节点处理：计算最长公共前缀
        commonPrefix := longestPrefix(search, n.prefix)
        
        if commonPrefix == len(n.prefix) {
            // 6.1 公共前缀等于节点前缀，继续深入
            search = search[commonPrefix:]
            continue
        }

        // 6.2 需要拆分节点
        child := &node{
            typ:    ntStatic,
            prefix: search[:commonPrefix],
        }
        parent.replaceChild(search[0], segTail, child)

        // 6.3 原节点作为新节点的子节点
        n.label = n.prefix[commonPrefix]
        n.prefix = n.prefix[commonPrefix:]
        child.addChild(n, n.prefix)

        // 6.4 新 key 是公共前缀的子集，直接设置为叶子
        search = search[commonPrefix:]
        if len(search) == 0 {
            child.setEndpoint(method, handler, pattern)
            return child
        }

        // 6.5 创建新的边
        subchild := &node{
            typ:    ntStatic,
            label:  search[0],
            prefix: search,
        }
        hn := child.addChild(subchild, search)
        hn.setEndpoint(method, handler, pattern)
        return hn
    }
}
```
**[tree.go:139-228](tree.go#L139-L228)**

### 3.3 节点拆分示例

假设有以下插入顺序：
1. 插入 `/contact`
2. 插入 `/continue`

插入过程：
1. 插入 `/contact`：根节点直接创建子节点 `prefix="contact"`
2. 插入 `/continue`：
   - 计算公共前缀：`"cont"`（4 个字符）
   - 拆分原节点：创建新节点 `prefix="cont"`
   - 原节点变为 `prefix="act"`，label='a'
   - 新节点 `prefix="inue"`，label='i'

结果树结构：
```
root
  └── cont (static)
        ├── act (static) → /contact
        └── inue (static) → /continue
```

### 3.4 addChild：处理参数节点

当路由包含参数时，`addChild` 方法负责递归拆分：

```go
func (n *node) addChild(child *node, prefix string) *node {
    search := prefix
    hn := child

    // 解析下一段
    segTyp, _, segRexpat, segTail, segStartIdx, segEndIdx := patNextSegment(search)

    switch segTyp {
    case ntStatic:
        // 纯静态，不做特殊处理

    default:
        // 处理参数/正则/通配符
        if segTyp == ntRegexp {
            rex, err := regexp.Compile(segRexpat)
            // ... 编译正则
            child.prefix = segRexpat
            child.rex = rex
        }

        if segStartIdx == 0 {
            // 路由以参数开头，如 /{id}/edit
            child.typ = segTyp
            child.tail = segTail

            if segStartIdx != len(search) {
                // 参数后还有内容，递归添加静态边
                search = search[segStartIdx:]
                nn := &node{
                    typ:    ntStatic,
                    label:  search[0],
                    prefix: search,
                }
                hn = child.addChild(nn, search)
            }
        } else if segStartIdx > 0 {
            // 路由以静态开头，如 /article/{id}
            child.typ = ntStatic
            child.prefix = search[:segStartIdx]

            // 添加参数节点作为子节点
            search = search[segStartIdx:]
            nn := &node{
                typ:   segTyp,
                label: search[0],
                tail:  segTail,
            }
            hn = child.addChild(nn, search)
        }
    }

    // 将节点添加到对应类型的数组中并排序
    n.children[child.typ] = append(n.children[child.typ], child)
    n.children[child.typ].Sort()
    return hn
}
```
**[tree.go:235-317](tree.go#L235-L317)**

## 4. 路由匹配流程

### 4.1 入口：Mux.routeHTTP()

请求进入时的处理流程：

```go
func (mx *Mux) routeHTTP(w http.ResponseWriter, r *http.Request) {
    rctx := r.Context().Value(RouteCtxKey).(*Context)

    // 1. 获取请求路径
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

    // 2. 转换方法类型
    if rctx.RouteMethod == "" {
        rctx.RouteMethod = r.Method
    }
    method, ok := methodMap[rctx.RouteMethod]
    if !ok {
        mx.MethodNotAllowedHandler().ServeHTTP(w, r)
        return
    }

    // 3. 在 Radix Tree 中查找路由
    if _, _, h := mx.tree.FindRoute(rctx, method, routePath); h != nil {
        // 3.1 设置路径参数到 request
        for i, key := range rctx.URLParams.Keys {
            value := rctx.URLParams.Values[i]
            r.SetPathValue(key, value)
        }
        r.Pattern = rctx.routePattern
        
        // 3.2 执行 handler
        h.ServeHTTP(w, r)
        return
    }

    // 4. 处理 405 或 404
    if rctx.methodNotAllowed {
        mx.MethodNotAllowedHandler(rctx.methodsAllowed...).ServeHTTP(w, r)
    } else {
        mx.NotFoundHandler().ServeHTTP(w, r)
    }
}
```
**[mux.go:441-485](mux.go#L441-L485)**

### 4.2 核心匹配算法：findRoute()

这是整个路由系统最核心的算法，决定了匹配顺序和策略：

```go
func (n *node) findRoute(rctx *Context, method methodTyp, path string) *node {
    nn := n
    search := path

    // 按类型顺序遍历子节点数组
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
            // ========== 静态节点匹配 ==========
            // 使用二分查找定位节点
            xn = nds.findEdge(label)
            if xn == nil || !strings.HasPrefix(xsearch, xn.prefix) {
                continue
            }
            xsearch = xsearch[len(xn.prefix):]

        case ntParam, ntRegexp:
            // ========== 参数/正则节点匹配 ==========
            // 空路径直接跳过
            if xsearch == "" {
                continue
            }

            // 顺序遍历每个同类型节点（按 tail 分组）
            for _, xn = range nds {
                // 查找分隔符位置
                p := strings.IndexByte(xsearch, xn.tail)

                if p < 0 {
                    // 没有找到分隔符
                    if xn.tail == '/' {
                        // 如果 tail 是 '/', 则匹配到路径末尾
                        p = len(xsearch)
                    } else {
                        continue
                    }
                } else if ntyp == ntRegexp && p == 0 {
                    // 正则节点不能匹配空字符串
                    continue
                }

                // 正则表达式验证
                if ntyp == ntRegexp && xn.rex != nil {
                    if !xn.rex.MatchString(xsearch[:p]) {
                        continue
                    }
                } else if strings.IndexByte(xsearch[:p], '/') != -1 {
                    // 普通参数不能跨路径段（不能包含 '/')
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
                            rctx.routeParams.Keys = append(rctx.routeParams.Keys, h.paramKeys...)
                            return xn
                        }
                        // 记录允许的方法（用于 405）
                        for endpoints := range xn.endpoints {
                            if endpoints == mALL || endpoints == mSTUB {
                                continue
                            }
                            rctx.methodsAllowed = append(rctx.methodsAllowed, endpoints)
                        }
                        rctx.methodNotAllowed = true
                    }
                }

                // 递归深入匹配
                fin := xn.findRoute(rctx, method, xsearch)
                if fin != nil {
                    return fin
                }

                // 回溯：恢复参数状态
                rctx.routeParams.Values = rctx.routeParams.Values[:prevlen]
                xsearch = search
            }

            // 参数节点未找到匹配时的空参数占位
            rctx.routeParams.Values = append(rctx.routeParams.Values, "")

        default:
            // ========== 通配符节点 (ntCatchAll) ==========
            // 捕获剩余全部路径
            rctx.routeParams.Values = append(rctx.routeParams.Values, search)
            xn = nds[0]
            xsearch = ""
        }

        if xn == nil {
            continue
        }

        // 检查是否找到完整匹配
        if len(xsearch) == 0 {
            if xn.isLeaf() {
                h := xn.endpoints[method]
                if h != nil && h.handler != nil {
                    rctx.routeParams.Keys = append(rctx.routeParams.Keys, h.paramKeys...)
                    return xn
                }
                // 记录允许的方法
                for endpoints := range xn.endpoints {
                    if endpoints == mALL || endpoints == mSTUB {
                        continue
                    }
                    rctx.methodsAllowed = append(rctx.methodsAllowed, endpoints)
                }
                rctx.methodNotAllowed = true
            }
        }

        // 递归深入
        fin := xn.findRoute(rctx, method, xsearch)
        if fin != nil {
            return fin
        }

        // 回溯：移除当前节点类型的参数
        if xn.typ > ntStatic {
            if len(rctx.routeParams.Values) > 0 {
                rctx.routeParams.Values = rctx.routeParams.Values[:len(rctx.routeParams.Values)-1]
            }
        }
    }

    return nil
}
```
**[tree.go:401-544](tree.go#L401-L544)**

### 4.3 匹配优先级

**关键结论**：匹配顺序由 `children` 数组的遍历顺序决定：

```
优先级从高到低：
1. ntStatic (静态路由)    → children[0]
2. ntRegexp (正则路由)    → children[1]
3. ntParam (参数路由)     → children[2]
4. ntCatchAll (通配符路由) → children[3]
```

#### 示例说明

假设有以下路由注册：

```go
r.Get("/article/search", searchHandler)       // 静态
r.Get("/article/{id:[0-9]+}", idHandler)     // 正则
r.Get("/article/{slug}", slugHandler)         // 参数
r.Get("/article/*", catchAllHandler)          // 通配符
```

请求 `GET /article/search`：
1. 首先匹配静态节点 `/article/search` ✓ → 返回 `searchHandler`

请求 `GET /article/123`：
1. 静态节点无匹配
2. 正则节点 `{id:[0-9]+}` 匹配 "123" ✓ → 返回 `idHandler`

请求 `GET /article/hello`：
1. 静态节点无匹配
2. 正则节点不匹配（"hello" 不是数字）
3. 参数节点 `{slug}` 匹配 ✓ → 返回 `slugHandler`

请求 `GET /article/a/b/c`：
1. 静态、正则、参数节点都不匹配（参数不能跨 `/`）
2. 通配符节点匹配 ✓ → 返回 `catchAllHandler`，`*` = "a/b/c"

### 4.4 同类型节点的匹配顺序

在同一类型的节点中：
- **静态节点**：按 `label` 排序，使用二分查找
- **参数/正则节点**：
  1. 按 `label` 排序（通常都是 `'{'`）
  2. `tailSort()` 将 `tail='/'` 的节点移到**最后**

这意味着：非 `/` 结尾的参数节点优先匹配。

#### 示例

```go
r.Get("/articles/{id}:delete", deleteHandler)  // tail = ':'
r.Get("/articles/{id}/edit", editHandler)      // tail = '/'
```

请求 `/articles/123:delete`：
- 优先匹配 `tail=':'` 的节点 → `deleteHandler`

请求 `/articles/123/edit`：
- `tail=':'` 不匹配（路径中没有 `:`）
- 然后匹配 `tail='/'` 的节点 → `editHandler`

## 5. 冲突检测机制

chi 的冲突检测分为几个层次：

### 5.1 Mount 时的冲突检测

在 `Mount` 子路由时，会显式检查是否已有冲突的路由：

```go
func (mx *Mux) Mount(pattern string, handler http.Handler) {
    // ...
    
    // 检查是否已有相同路径的 mount
    if mx.tree.findPattern(pattern+"*") || mx.tree.findPattern(pattern+"/*") {
        panic(fmt.Sprintf("chi: attempting to Mount() a handler on an existing path, '%s'", pattern))
    }
    
    // ...
}
```
**[mux.go:295-298](mux.go#L295-L298)**

**检测逻辑**：
- `findPattern(pattern+"*")` 检查是否已有 `pattern*` 形式的路由
- `findPattern(pattern+"/*")` 检查是否已有 `pattern/*` 形式的路由

如果已存在，直接 `panic`。

### 5.2 重复参数名检测

在 `patParamKeys` 函数中检测同一路由中是否有重复的参数名：

```go
func patParamKeys(pattern string) []string {
    pat := pattern
    paramKeys := []string{}
    for {
        ptyp, paramKey, _, _, _, e := patNextSegment(pat)
        if ptyp == ntStatic {
            return paramKeys
        }
        // 检查重复的参数名
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
**[tree.go:755-771](tree.go#L755-L771)**

例如：`/users/{id}/posts/{id}` 会 panic，因为参数名 `id` 重复。

### 5.3 通配符位置检测

在 `patNextSegment` 中检测通配符的位置：

```go
func patNextSegment(pattern string) (nodeTyp, string, string, byte, int, int) {
    ps := strings.Index(pattern, "{")
    ws := strings.Index(pattern, "*")

    // 通配符必须在参数之后
    if ps >= 0 && ws >= 0 && ws < ps {
        panic("chi: wildcard '*' must be the last pattern in a route, otherwise use a '{param}'")
    }

    // ...

    // 通配符必须是路径的最后一段
    if ws < len(pattern)-1 {
        panic("chi: wildcard '*' must be the last value in a route. trim trailing text or use a '{param}' instead")
    }
    return ntCatchAll, "*", "", 0, ws, len(pattern)
}
```
**[tree.go:687-753](tree.go#L687-L753)**

非法路由示例：
- `/api/*/users` → panic（通配符后还有内容）
- `/*/{id}` → panic（通配符在参数之前）

### 5.4 同方法同路径的覆盖

**注意**：对于普通路由（非 Mount），chi **不会检测冲突**，而是**后注册的覆盖先注册的**。

从测试代码可以看到：

```go
tr.InsertRoute(mGET, "/article/{id}", hStub)
tr.InsertRoute(mGET, "/article/{id}", hArticleShow)  // 覆盖上面的
tr.InsertRoute(mGET, "/article/{id}", hArticleShow)  // 重复，无效果（还是这个）
```
**[tree_test.go:44-46](tree_test.go#L44-L46)**

`setEndpoint` 方法只是简单地赋值：

```go
func (n *node) setEndpoint(method methodTyp, handler http.Handler, pattern string) {
    // ...
    h := n.endpoints.Value(method)
    h.handler = handler  // 直接覆盖
    h.pattern = pattern
    h.paramKeys = paramKeys
    // ...
}
```
**[tree.go:344-372](tree.go#L344-L372)**

### 5.5 冲突检测层次总结

| 层次 | 检测内容 | 触发方式 |
|------|----------|----------|
| 语法层 | 通配符位置、参数闭合 | `patNextSegment` 解析时 panic |
| 参数层 | 重复参数名 | `patParamKeys` 检测时 panic |
| Mount 层 | 重复挂载路径 | `findPattern` 检测时 panic |
| 普通路由 | 同方法同路径 | **无检测，直接覆盖** |

## 6. 数据结构与算法总结

### 6.1 核心数据结构

```
node (根节点)
│
├── children[0] → []*node (静态节点列表，按 label 排序，二分查找)
│       ├── node{prefix: "/home", typ: ntStatic, ...}
│       └── node{prefix: "/article", typ: ntStatic, ...}
│
├── children[1] → []*node (正则节点列表，按 label+tail 排序)
│       └── node{prefix: "^[0-9]+", typ: ntRegexp, rex: regexp, tail: '/', ...}
│
├── children[2] → []*node (参数节点列表，tail='/' 移到最后)
│       ├── node{typ: ntParam, tail: ':', ...}
│       └── node{typ: ntParam, tail: '/', ...}
│
└── children[3] → []*node (通配符节点列表，通常只有一个)
        └── node{typ: ntCatchAll, ...}
```

### 6.2 算法复杂度

| 操作 | 静态节点 | 参数/正则/通配符 |
|------|----------|------------------|
| **查找** | O(log n) 二分查找 | O(k) 顺序遍历（k 为同类型节点数） |
| **插入** | O(log n) + 节点拆分 | O(k) + 递归创建 |

**实际性能**：
- 静态路由占大多数时，性能接近 O(log n)
- 参数节点数量通常很少，顺序遍历可忽略
- Radix Tree 的路径压缩大幅减少了节点数量

### 6.3 设计亮点

1. **按类型分组的 children 数组**：
   - 天然决定了匹配优先级
   - 类型隔离避免了不同类型节点的互相干扰

2. **tail 分隔符机制**：
   - 支持非 `/` 分隔的参数，如 `/{id}.json`、`/{id}:edit`
   - `tailSort` 确保更具体的分隔符优先匹配

3. **回溯式参数收集**：
   - `findRoute` 在递归时记录参数
   - 匹配失败时回溯（恢复 `rctx.routeParams.Values`）
   - 支持多分支探索

4. **方法允许检测**：
   - 找到节点但方法不匹配时，记录 `methodsAllowed`
   - 用于返回 `405 Method Not Allowed` 和 `Allow` 响应头

## 7. 示例：完整路由树构建

假设有以下路由注册：

```go
r := chi.NewRouter()
r.Get("/", homeHandler)
r.Get("/articles", articlesListHandler)
r.Get("/articles/{id:[0-9]+}", articleDetailHandler)
r.Get("/articles/{slug}", articleSlugHandler)
r.Get("/articles/{id}/edit", articleEditHandler)
r.Get("/admin/*", adminCatchAllHandler)
```

构建后的树结构（简化）：

```
root (typ=ntStatic, prefix="", endpoints=nil)
│
├── children[0] (static nodes)
│       └── node{label='/', prefix="/", ...}
│               │
│               ├── children[0] (static)
│               │       ├── node{prefix="articles", endpoints={GET: articlesListHandler}}
│               │       │       │
│               │       │       ├── children[0] (static)
│               │       │       │       └── node{prefix="/edit", ...}
│               │       │       │
│               │       │       ├── children[1] (regexp)
│               │       │       │       └── node{typ=ntRegexp, tail='/', rex=/^[0-9]+$/, ...}
│               │       │       │               ├── children[0] (static)
│               │       │       │               │       └── node{prefix="/edit", endpoints={GET: articleEditHandler}}
│               │       │       │               └── endpoints = {GET: articleDetailHandler}
│               │       │       │
│               │       │       └── children[2] (param)
│               │       │               └── node{typ=ntParam, tail='/', endpoints={GET: articleSlugHandler}}
│               │       │
│               │       └── node{prefix="admin", ...}
│               │               │
│               │               └── children[3] (catchAll)
│               │                       └── node{typ=ntCatchAll, endpoints={GET: adminCatchAllHandler}}
│               │
│               └── endpoints = {GET: homeHandler}
│
├── children[1] (regexp nodes: 空)
├── children[2] (param nodes: 空)
└── children[3] (catchAll nodes: 空)
```

## 8. 总结

go-chi 的路由系统设计精巧，核心要点：

1. **Radix Tree + 类型分组**：
   - 使用基数树实现高效的前缀匹配
   - 按 `ntStatic` → `ntRegexp` → `ntParam` → `ntCatchAll` 顺序分组存储和匹配

2. **节点类型区分**：
   - 通过 `node.typ` 字段区分四种节点类型
   - 参数/正则节点额外使用 `tail` 字段标识分隔符

3. **匹配顺序**：
   - 静态优先于正则，正则优先于参数，参数优先于通配符
   - 同类型中，非 `/` 分隔的参数优先匹配

4. **冲突检测**：
   - Mount 时严格检测重复路径
   - 语法层面检测通配符位置和参数闭合
   - **普通路由同方法同路径直接覆盖，无冲突检测**

5. **回溯机制**：
   - 参数收集采用回溯策略
   - 支持多分支探索，找到最匹配的路由

这种设计在性能、灵活性和可维护性之间取得了良好的平衡，是 chi 成为流行 Go Web 框架的重要原因之一。
