+++
title = "Gin源码随读"
date = 2020-12-10T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

使用 gin 框架最常用的方式是使用 `gin.New` 或者 `gin.Default` 函数创建一个 `*gin.Engine` 实例，然后使用实例的方法注册 HTTP 路由函数，最后调用 `Run` 方法监听端口和启动服务。

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

现在来窥探这背后发生了什么：

`Engine` 对象是 gin 的框架实例，它其中包括了路由定义以及一些配置相关的参数：

- `RouterGroup`：管理路由和中间件的组件，它定义了 URL 路径与处理函数的映射关系。
- `RedirectTrailingSlash`：如果当前路径的处理函数不存在，但是路径+'/'的处理函数存在，则允许进行重定向，默认为 true。
- `RedirectFixedPath`：允许修复当前请求路径，如/FOO和/..//Foo会被修复为/foo，并进行重定向，默认为 false。
- `UseRawPath`：使用未转义的请求路径（url.RawPath），默认为 false。
- `UnescapePathValues`：对请求路径值进行转义（url.Path），默认为 true。
- `RemoveExtraSlash`：去除额外的反斜杠，默认为 false。
- `trees`：每一个 HTTP 方法会有一颗方法树，方法树记录了路径和路径上的处理函数。

```go
type Engine struct {
	RouterGroup

	// Enables automatic redirection if the current route can't be matched but a
	// handler for the path with (without) the trailing slash exists.
	// For example if /foo/ is requested but a route only exists for /foo, the
	// client is redirected to /foo with http status code 301 for GET requests
	// and 307 for all other request methods.
	RedirectTrailingSlash bool

	// If enabled, the router tries to fix the current request path, if no
	// handle is registered for it.
	// First superfluous path elements like ../ or // are removed.
	// Afterwards the router does a case-insensitive lookup of the cleaned path.
	// If a handle can be found for this route, the router makes a redirection
	// to the corrected path with status code 301 for GET requests and 307 for
	// all other request methods.
	// For example /FOO and /..//Foo could be redirected to /foo.
	// RedirectTrailingSlash is independent of this option.
	RedirectFixedPath bool

	// If enabled, the router checks if another method is allowed for the
	// current route, if the current request can not be routed.
	// If this is the case, the request is answered with 'Method Not Allowed'
	// and HTTP status code 405.
	// If no other Method is allowed, the request is delegated to the NotFound
	// handler.
	HandleMethodNotAllowed bool
	ForwardedByClientIP    bool

	// #726 #755 If enabled, it will thrust some headers starting with
	// 'X-AppEngine...' for better integration with that PaaS.
	AppEngine bool

	// If enabled, the url.RawPath will be used to find parameters.
	UseRawPath bool

	// If true, the path value will be unescaped.
	// If UseRawPath is false (by default), the UnescapePathValues effectively is true,
	// as url.Path gonna be used, which is already unescaped.
	UnescapePathValues bool

	// Value of 'maxMemory' param that is given to http.Request's ParseMultipartForm
	// method call.
	MaxMultipartMemory int64

	// RemoveExtraSlash a parameter can be parsed from the URL even with extra slashes.
	// See the PR #1817 and issue #1644
	RemoveExtraSlash bool

	delims           render.Delims
	secureJsonPrefix string
	HTMLRender       render.HTMLRender
	FuncMap          template.FuncMap
	allNoRoute       HandlersChain
	allNoMethod      HandlersChain
	noRoute          HandlersChain
	noMethod         HandlersChain
	pool             sync.Pool
	trees            methodTrees
}
```

`RouterGroup` 用来配置 HTTP 路由，它关联了一个路径前缀和其对应的处理函数，同时 `RouterGroup` 也包含了关联它的 `Engine` 对象，当调用 `RouterGroup` 的路由定义方法时会在 `Engine` 的路由树上创建路径与其处理函数。

```go
type RouterGroup struct {
	Handlers HandlersChain
	basePath string
	engine   *Engine
	root     bool
}
```

`HandlerFunc` 是路由的处理函数，它在 gin 中的定义如下，是一个接收 `*Context` 作为参数的函数，`HandlersChain` 是处理函数的调用链，通常包括了路由上定义的中间件以及最终处理函数。

```go
// HandlerFunc defines the handler used by gin middleware as return value.
type HandlerFunc func(*Context)

// HandlersChain defines a HandlerFunc array.
type HandlersChain []HandlerFunc

// Last returns the last handler in the chain. ie. the last handler is the main one.
func (c HandlersChain) Last() HandlerFunc {
	if length := len(c); length > 0 {
		return c[length-1]
	}
	return nil
}
```

`Context` 是处理函数调用链传递的对象，它包括了 HTTP 的请求对象，请求参数，和构造 HTTP 响应的对象，它允许使用者在调用链中传递自定义变量，并在调用链的其它地方通过 `Context` 对象把它取出来。

```go
// Context is the most important part of gin. It allows us to pass variables between middleware,
// manage the flow, validate the JSON of a request and render a JSON response for example.
type Context struct {
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter

	Params   Params
	handlers HandlersChain
	index    int8
	fullPath string

	engine *Engine

	// This mutex protect Keys map
	KeysMutex *sync.RWMutex

	// Keys is a key/value pair exclusively for the context of each request.
	Keys map[string]interface{}

	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs

	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string

	// queryCache use url.ParseQuery cached the param query result from c.Request.URL.Query()
	queryCache url.Values

	// formCache use url.ParseQuery cached PostForm contains the parsed form data from POST, PATCH,
	// or PUT body parameters.
	formCache url.Values

	// SameSite allows a server to define a cookie attribute making it impossible for
	// the browser to send this cookie along with cross-site requests.
	sameSite http.SameSite
}
```

`Context` 还记录了当前调用链的 `index`，对应着处理函数在调用链中的位置，并可通过调用 `Next` 方法继续执行调用链中的下一个处理函数，来实现调用链的控制流。

![gin_handler_chain](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/gin_handler_chain.png)

```go
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
// See example in GitHub.
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}
```

`RouterGroup` 实现了 `IRouter` 接口定义的方法，因此 `Engine` 也是一个 `IRouter` 的实现，`IRouter` 定义了 HTTP 路由的创建方法，包括`GET`，`POST`，`PUT`，`DELETE`方法等，以及 `Group` 方法，它会返回一个`*RouterGroup`对象，用来创建一个路由组。

```go
// IRouter defines all router handle interface includes single and group router.
type IRouter interface {
	IRoutes
	Group(string, ...HandlerFunc) *RouterGroup
}

// IRoutes defines all router handle interface.
type IRoutes interface {
	Use(...HandlerFunc) IRoutes

	Handle(string, string, ...HandlerFunc) IRoutes
	Any(string, ...HandlerFunc) IRoutes
	GET(string, ...HandlerFunc) IRoutes
	POST(string, ...HandlerFunc) IRoutes
	DELETE(string, ...HandlerFunc) IRoutes
	PATCH(string, ...HandlerFunc) IRoutes
	PUT(string, ...HandlerFunc) IRoutes
	OPTIONS(string, ...HandlerFunc) IRoutes
	HEAD(string, ...HandlerFunc) IRoutes

	StaticFile(string, string) IRoutes
	Static(string, string) IRoutes
	StaticFS(string, http.FileSystem) IRoutes
}
```

当 `Engine` 在执行 `Run` 方法时，会调用 `resolveAddress` 解析传入的地址，若没有传入地址，则默认使用 `PORT` 环境变量作为端口号，在此端口上运行 HTTP 服务，接着调用 `http.ListenAndServe` ，把自身作为一个 `http.Handler`，监听和处理 HTTP 请求。

```go
// Run attaches the router to a http.Server and starts listening and serving HTTP requests.
// It is a shortcut for http.ListenAndServe(addr, router)
// Note: this method will block the calling goroutine indefinitely unless an error happens.
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine)
	return
}
```

`Engine` 实现了 `http.Handler` 的 `ServeHTTP(w http.ResponseWriter, req *http.Request)` 方法，接收到 HTTP 请求后会统一走到 `ServeHTTP` 方法中，首先 `Engine` 会从 context 对象池中取出一个 `*Context`，使用对象池管理  `Context`  可以尽量减少频繁创建对象带来的 GC，拿出一个 `*Context` 之后，它就会作为请求的上下文，传递到请求方法里。

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```

在 `Engine` 的 `handleHTTPRequest` 方法中，进行了下面几个步骤：

1. 根据配置参数决定是否使用编码后的 URL 路径，以及去除多余的反斜杠。
2. 根据 HTTP 请求方法找到对应的方法树，若找到对应的方法树，从方法树中获取路由信息，并把处理函数，参数路径信息记录到 `Context` 上，调用 `Context` 的 `Next` 方法开始执行调用链上的函数。若方法树中不存在路由信息，则判断路径+'/'的路由定义是否存在，并尝试进行重定向。
3. 如果没有找到对应路由信息，根据配置参数返回 HTTP 404 (NOT FOUND) 或 405 (METHOD NOT ALLOWED) 错误。

```go
func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method
	rPath := c.Request.URL.Path
	unescape := false
	if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
		rPath = c.Request.URL.RawPath
		unescape = engine.UnescapePathValues
	}

	if engine.RemoveExtraSlash {
		rPath = cleanPath(rPath)
	}

	// Find root of the tree for the given HTTP method
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// Find route in tree
		value := root.getValue(rPath, c.Params, unescape)
		if value.handlers != nil {
			c.handlers = value.handlers
			c.Params = value.params
			c.fullPath = value.fullPath
			c.Next()
			c.writermem.WriteHeaderNow()
			return
		}
		if httpMethod != "CONNECT" && rPath != "/" {
			if value.tsr && engine.RedirectTrailingSlash {
				redirectTrailingSlash(c)
				return
			}
			if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
				return
			}
		}
		break
	}

	if engine.HandleMethodNotAllowed {
		for _, tree := range engine.trees {
			if tree.method == httpMethod {
				continue
			}
			if value := tree.root.getValue(rPath, nil, unescape); value.handlers != nil {
				c.handlers = engine.allNoMethod
				serveError(c, http.StatusMethodNotAllowed, default405Body)
				return
			}
		}
	}
	c.handlers = engine.allNoRoute
	serveError(c, http.StatusNotFound, default404Body)
}
```

在理解怎么从一颗方法树上找到路由信息前，先了解如何在一颗方法树上创建路径节点，以创建一个 GET 方法的路由为例，当在调用 Router 的 `GET` 方法时，最终会调用到 `Engine` 的 `addRoute` 方法，`addRoute` 会根据 HTTP 方法获取到对应的方法树，如果方法树不存在，则创建一颗方法树，添加到 `Engine` 的 `trees` 上，最后在方法树上调用 `addRoute` 创建路径节点。

```go
// GET is a shortcut for router.Handle("GET", path, handle).
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodGet, relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}

// ============== gin.go ===============
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	assert1(path[0] == '/', "path must begin with '/'")
	assert1(method != "", "HTTP method can not be empty")
	assert1(len(handlers) > 0, "there must be at least one handler")

	debugPrintRoute(method, path, handlers)
	root := engine.trees.get(method)
	if root == nil {
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
	root.addRoute(path, handlers)
}
```

gin 的路由查找是基于 httprouter 实现的，它的方法树是一颗压缩前缀树，如下所示，是空间优化版本的前缀树，它会寻找路径相同的前缀，在相同前缀处产生分裂，把相同前缀作为父节点，而分裂处后的路径则作为父节点的子节点，比如对于 `search` 和 `support`，它会分裂为  `s`，`earch` 和 `upport` 三个节点，其中 `s` 为父节点，其余两者则是它的分别两个子节点。

![radix-tree](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/radix-tree.png)

树的数据结构如下，它们的作用如下：

- `path`：当前节点路径的值。（路径部分前缀）
- `fullpath`：当前节点完全路径的值。（路径完整前缀）
- `wildChild`：当前节点是否有一个带参数的子节点。
- `priority`：当前节点的权重值，如果当前节点底下的处理函数越多，则它的值越大，优先被匹配到。
- `children`：当前节点下的子节点列表。
- `indices`：包含子节点首字符的前缀索引，比如 `r` 下游两个子节点 `om` 和 `ub`，则 `r` 节点的 `indices` 为 `ou`。
- `nType`：节点类型，可为`static`， `root`，`param` 或 `catchAll`。

```go
type node struct {
	path      string
	indices   string
	children  []*node
	handlers  HandlersChain
	priority  uint32
	nType     nodeType
	maxParams uint8
	wildChild bool
	fullPath  string
}
```

再来看方法树上的 `addRoute` 方法，对于一个要在树上添加的路径，会进行以下步骤：

1. 如果树上不存在任何节点，则把当前节点作为根节点，插入到方法树上，节点路径为传入路径。
2. 否则，遍历树的节点：
   1. 计算当前节点的路径和传入路径的最长公共前缀位置。（比如已存在一个节点路径为 `/search/`，传入路径 `/support/` 与它的最长公共前缀位置是 2，意味着它们具有共同前缀`/s`）
   2. 如果公共前缀的长度小于当前节点的长度（`/s` 长度小于 `/search` ），则在当前节点产生分裂，生成一个路径为  `earch/` 的子节点，把它添加到当前节点的 `children`，并把首字符 `e` 添加到当前节点的前缀索引 `indices` 中，将当前节点的路径改为前缀路径（从 `/search/` 变为 `/s`）。
   3. 如果公共前缀的长度小于传入节点的长度（`/s` 长度小于 `/support` ），则在传入路径中产生一个新的路径（`upport/`），插入到当前节点的 `children`，把首字符 `u` 添加到当前节点的前缀索引 `indices` 中。
      1. 这里存在一种情况是，如果当前节点的子节点是一个参数节点（当前节点的`wildChild` 为 true），那么会检查传入路径是否也是相同的参数节点下的路径，比如当前节点路径为 `/user/:user_id`，传入节点路径为 `/user/:user_id/name`，如果满足条件的话，则继续到子节点（`:user_id`) 下创建新的路径，否则若在参数节点下定义了其他路径，如`/user/name`，则会直接发生 panic 返回，因为当前路径下存在冲突（一个参数节点不能跟一个非参数节点位于同级）。
      2. 如果当前节点是一个参数节点，（如 `:user_id`，在此节点下创建路径为 `/name`），并且路径以 `/` 开头且当前节点只存在一个子节点，则当前节点指向子节点，继续进行路径分裂。
      3. 如果当前节点存在多个子节点，则从 `indices` 查找匹配路径首字符的子节点，继续往子节点遍历。
      4. 否则直接往当前节点上创建子节点。（例如：定义路由为 `/user/:user_id/`，则 `:user_id` 会存在一个子节点为 `/`，这时候 `/name` 就需要跟 `/` 节点进行路径分裂插入到 `:user_id` 下，如果定义路由为 `/user/:user_id`，则直接插入到 `:user_id` 下就好了）

```go
// addRoute adds a node with the given handle to the path.
// Not concurrency-safe!
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path
	n.priority++
	numParams := countParams(path)

	// Empty tree
	if len(n.path) == 0 && len(n.children) == 0 {
		n.insertChild(numParams, path, fullPath, handlers)
		n.nType = root
		return
	}

	parentFullPathIndex := 0

walk:
	for {
		// Update maxParams of the current node
		if numParams > n.maxParams {
			n.maxParams = numParams
		}

		// Find the longest common prefix.
		// This also implies that the common prefix contains no ':' or '*'
		// since the existing key can't contain those chars.
		i := longestCommonPrefix(path, n.path)

		// Split edge
		if i < len(n.path) {
			child := node{
				path:      n.path[i:],
				wildChild: n.wildChild,
				indices:   n.indices,
				children:  n.children,
				handlers:  n.handlers,
				priority:  n.priority - 1,
				fullPath:  n.fullPath,
			}

			// Update maxParams (max of all children)
			for _, v := range child.children {
				if v.maxParams > child.maxParams {
					child.maxParams = v.maxParams
				}
			}

			n.children = []*node{&child}
			// []byte for proper unicode char conversion, see #65
			n.indices = string([]byte{n.path[i]})
			n.path = path[:i]
			n.handlers = nil
			n.wildChild = false
			n.fullPath = fullPath[:parentFullPathIndex+i]
		}

		// Make new node a child of this node
		if i < len(path) {
			path = path[i:]

			if n.wildChild {
				parentFullPathIndex += len(n.path)
				n = n.children[0]
				n.priority++

				// Update maxParams of the child node
				if numParams > n.maxParams {
					n.maxParams = numParams
				}
				numParams--

				// Check if the wildcard matches
				if len(path) >= len(n.path) && n.path == path[:len(n.path)] {
					// check for longer wildcard, e.g. :name and :names
					if len(n.path) >= len(path) || path[len(n.path)] == '/' {
						continue walk
					}
				}

				pathSeg := path
				if n.nType != catchAll {
					pathSeg = strings.SplitN(path, "/", 2)[0]
				}
				prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
				panic("'" + pathSeg +
					"' in new path '" + fullPath +
					"' conflicts with existing wildcard '" + n.path +
					"' in existing prefix '" + prefix +
					"'")
			}

			c := path[0]

			// slash after param
			if n.nType == param && c == '/' && len(n.children) == 1 {
				parentFullPathIndex += len(n.path)
				n = n.children[0]
				n.priority++
				continue walk
			}

			// Check if a child with the next path byte exists
			for i, max := 0, len(n.indices); i < max; i++ {
				if c == n.indices[i] {
					parentFullPathIndex += len(n.path)
					i = n.incrementChildPrio(i)
					n = n.children[i]
					continue walk
				}
			}

			// Otherwise insert it
			if c != ':' && c != '*' {
				// []byte for proper unicode char conversion, see #65
				n.indices += string([]byte{c})
				child := &node{
					maxParams: numParams,
					fullPath:  fullPath,
				}
				n.children = append(n.children, child)
				n.incrementChildPrio(len(n.indices) - 1)
				n = child
			}
			n.insertChild(numParams, path, fullPath, handlers)
			return
		}

		// Otherwise and handle to current node
		if n.handlers != nil {
			panic("handlers are already registered for path '" + fullPath + "'")
		}
		n.handlers = handlers
		return
	}
}
```

举一个例子来演示树的创建过程，如下所示，假设现在定义了一些 HTTP 路由函数：

```go
	r := gin.Default()
	r.GET("/search/", func(ctx *gin.Context){})
	r.GET("/support/", func(ctx *gin.Context){})
	r.GET("/blog/", func(ctx *gin.Context){})
	r.GET("/blog/:post/", func(ctx *gin.Context){})
	r.GET("/about-us/", func(ctx *gin.Context){})
	r.GET("/about-us/team/", func(ctx *gin.Context){})
	r.Run()
```

节点的创建过程如下：

首先只有一颗空树，这时候插入 `/search/`，它就变成了树的根节点：

![radix-tree-1](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/radix-tree-1.png)

接下来插入 `/support/`，因为它与 `/search/` 具有最长公共前缀 `/s`，所以此处产生路径分裂，生成两个子节点 `earch/` 以及 `upport/`，它们具有共同的父节点，即公共前缀 `/s`：

![radix-tree-2](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/radix-tree-2.png)

接着插入 `/blog/`，因为它与  `/s`  具有最长公共前缀  `/`，所以此处分裂为  `s` 和  `blog/` 两个子节点，父节点都为 `/`：

![radix-tree-3](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/radix-tree-3.png)

接下来插入 `/blog/:post/`，它与 `/` 具有最长公共前缀  `/`，因此要插入的路径为 `blog/:post/`，然后根据当前节点 `/` 的 `indices` 判断与路径的首字符 `b` 匹配的节点为 `blog/`，于是就把当前节点指向`blog/`，因为  `blog/` 下不存在子节点，所以在  `blog/` 下创建路径为 `:post` 的 wildcard 节点，因为 `:post/` 以 `/` 结尾，因此在 wildcard 节点下创建一个路径为 `/` 的子节点。

![radix-tree-4](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/radix-tree-4.png)

最后插入 `/about-us/` 和 `/about-us/team` 节点，最终路由树的结构如下：

![radix-tree-5](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/radix-tree-5.png)

现在在再回到怎么根据一个传入路径在一棵树上找到对应的路由信息，方法树的 `getValue` 方法定义如下，它主要进行了几个步骤：

1. 如果当前节点路径与传入路径相等
   1. 如果当前节点的处理函数不为空，结束并返回路由信息。
   2. 如果当前节点不存在处理函数，则尝试寻找路径+'/'上是否注册了处理函数，如果有则尝试进行重定向。
2. 如果当前节点路径与传入路径具有相同前缀
   1. 如果当前节点不存在一个带参数的子节点，则查找并遍历到下一个匹配的子节点。
   2. 否则如果当前节点存在一个带参数的子节点，则解析请求参数并记录到返回值的 `params` 里，如果路径后还有子路径（如：`/user/123/name`），则继续尝试匹配当前节点下的子节点，直至完全匹配返回。
3. 如果当前节点路径与传入路径不匹配，则尝试去寻找路径+"/"的节点是否存在，不存在则返回 HTTP 404 或 405 错误。

```go
// getValue returns the handle registered with the given path (key). The values of
// wildcards are saved to a map.
// If no handle can be found, a TSR (trailing slash redirect) recommendation is
// made if a handle exists with an extra (without the) trailing slash for the
// given path.
func (n *node) getValue(path string, po Params, unescape bool) (value nodeValue) {
	value.params = po
walk: // Outer loop for walking the tree
	for {
		prefix := n.path
		if path == prefix {
			// We should have reached the node containing the handle.
			// Check if this node has a handle registered.
			if value.handlers = n.handlers; value.handlers != nil {
				value.fullPath = n.fullPath
				return
			}

			if path == "/" && n.wildChild && n.nType != root {
				value.tsr = true
				return
			}

			// No handle found. Check if a handle for this path + a
			// trailing slash exists for trailing slash recommendation
			indices := n.indices
			for i, max := 0, len(indices); i < max; i++ {
				if indices[i] == '/' {
					n = n.children[i]
					value.tsr = (len(n.path) == 1 && n.handlers != nil) ||
						(n.nType == catchAll && n.children[0].handlers != nil)
					return
				}
			}

			return
		}

		if len(path) > len(prefix) && path[:len(prefix)] == prefix {
			path = path[len(prefix):]
			// If this node does not have a wildcard (param or catchAll)
			// child,  we can just look up the next child node and continue
			// to walk down the tree
			if !n.wildChild {
				c := path[0]
				indices := n.indices
				for i, max := 0, len(indices); i < max; i++ {
					if c == indices[i] {
						n = n.children[i]
						continue walk
					}
				}

				// Nothing found.
				// We can recommend to redirect to the same URL without a
				// trailing slash if a leaf exists for that path.
				value.tsr = path == "/" && n.handlers != nil
				return
			}

			// handle wildcard child
			n = n.children[0]
			switch n.nType {
			case param:
				// find param end (either '/' or path end)
				end := 0
				for end < len(path) && path[end] != '/' {
					end++
				}

				// save param value
				if cap(value.params) < int(n.maxParams) {
					value.params = make(Params, 0, n.maxParams)
				}
				i := len(value.params)
				value.params = value.params[:i+1] // expand slice within preallocated capacity
				value.params[i].Key = n.path[1:]
				val := path[:end]
				if unescape {
					var err error
					if value.params[i].Value, err = url.QueryUnescape(val); err != nil {
						value.params[i].Value = val // fallback, in case of error
					}
				} else {
					value.params[i].Value = val
				}

				// we need to go deeper!
				if end < len(path) {
					if len(n.children) > 0 {
						path = path[end:]
						n = n.children[0]
						continue walk
					}

					// ... but we can't
					value.tsr = len(path) == end+1
					return
				}

				if value.handlers = n.handlers; value.handlers != nil {
					value.fullPath = n.fullPath
					return
				}
				if len(n.children) == 1 {
					// No handle found. Check if a handle for this path + a
					// trailing slash exists for TSR recommendation
					n = n.children[0]
					value.tsr = n.path == "/" && n.handlers != nil
				}
				return

			case catchAll:
				// save param value
				if cap(value.params) < int(n.maxParams) {
					value.params = make(Params, 0, n.maxParams)
				}
				i := len(value.params)
				value.params = value.params[:i+1] // expand slice within preallocated capacity
				value.params[i].Key = n.path[2:]
				if unescape {
					var err error
					if value.params[i].Value, err = url.QueryUnescape(path); err != nil {
						value.params[i].Value = path // fallback, in case of error
					}
				} else {
					value.params[i].Value = path
				}

				value.handlers = n.handlers
				value.fullPath = n.fullPath
				return

			default:
				panic("invalid node type")
			}
		}

		// Nothing found. We can recommend to redirect to the same URL with an
		// extra trailing slash if a leaf exists for that path
		value.tsr = (path == "/") ||
			(len(prefix) == len(path)+1 && prefix[len(path)] == '/' &&
				path == prefix[:len(prefix)-1] && n.handlers != nil)
		return
	}
}
```

对于上面例子中创建的方法树，访问 `/blog/123/` 这个路径：

1. 因为 `/blog/123/` 与根节点 `/` 具有相同前缀 `/`，继续查找 `blog/123/` 发现它的首字符存在与根节点的 `indices` 上，移动当前节点到 `blog/` 上。

2. 因为 `blog/123/` 与 `blog/` 具有相同前缀 `blog/`，这时候继续在路径上查找 `123/` ，因为当前节点的子节点是一个参数节点（`:post`），所以移动当前节点到 `:post` 上，并进行请求参数的解析。因为 `123/` 的匹配参数的结束位置不是 `123/` 的末尾处，因此移动当前节点到 `/` 上，继续匹配 `/` 路径。

3. 这时候在 `/` 节点上发现路径跟节点前缀完全匹配，并且当前节点上注册了处理函数，因此返回匹配的路由信息。

如果访问的是 `/blog/123` 的话，则会出现首次会匹配失败（因为 `:post` 上不存在注册函数），但是这时候发现这个节点下存在一个子节点，于是会尝试给路径末尾加上一个反斜杠，变为 `/blog/123/`，再做一遍重定向，这时候就会再走一遍上面匹配的过程，就能匹配成功了。

![radix-tree-5](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/radix-tree-5.png)

当成功匹配到路由后，会调用 `Context` 的 `Next` 方法开始循环执行调用链上的函数，这时候一次完整的 HTTP 调用就结束了。

```go
// Find route in tree
value := root.getValue(rPath, c.Params, unescape)
if value.handlers != nil {
  c.handlers = value.handlers
  c.Params = value.params
  c.fullPath = value.fullPath
  c.Next()
  c.writermem.WriteHeaderNow()
  return
}
```


